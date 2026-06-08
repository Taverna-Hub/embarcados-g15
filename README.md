# WiFi MAC Capture e Análise de Sinal

Projeto embarcado para captura passiva de dispositivos WiFi próximos, orquestração dos dados via MQTT/Node-RED, análise com modulo de Machine Learning e exposição dos resultados por uma API FastAPI.

O fluxo principal do sistema é:

```text
ESP32 -> MQTT Mosquitto -> Node-RED -> Notebook/ML -> PostgreSQL -> Backend API
```

## Visão Geral

Este projeto identifica dispositivos próximos a partir de pacotes WiFi capturados passivamente. Os dados capturados pelo ESP32 são enviados para um broker MQTT, passam por validação e transformação no Node-RED, são analisados pelo serviço de ML/Notebook e gravados em um banco compartilhado. O backend consulta esse banco e disponibiliza endpoints REST para dispositivos, histórico e estatísticas.

## Módulos

### Hardware

Código embarcado do ESP32 responsável pela captura passiva de pacotes WiFi e publicação dos dados no tópico MQTT de entrada.

Arquivos principais:

- `hardware/src/main.cpp`
- `hardware/include/config.h`
- `hardware/platformio.ini`

### Node-RED e MQTT

O módulo `nodered/` atua como broker MQTT e orquestrador dos dados de captura.

Serviços iniciados via Docker Compose:

- Node-RED: `http://localhost:1880`
- Mosquitto MQTT Broker: `localhost:1883`

Tópicos principais:

- Entrada: `esp32/wifi/scan`
- Saída validada: `nodered/wifi/data`
- Erros: `nodered/errors`

O fluxo do Node-RED:

1. Recebe dados brutos do ESP32.
2. Faz parse do JSON.
3. Valida estrutura, MAC address, RSSI, canal e frequência.
4. Normaliza MAC addresses para uppercase.
5. Remove campos vazios e adiciona timestamp de processamento.
6. Publica os dados validados para o Notebook/ML.

### ML / Notebook

O módulo `ML/` processa os dados validados publicados pelo Node-RED e persiste os resultados no banco compartilhado.

Funcionalidades:

- Identificação de sistema operacional por MAC OUI.
- Estimativa de distância usando RSSI e frequência.
- Preservação de metadados da captura passiva, como canal, tipo de frame e quantidade de aparições.
- Classificação simples de localização como `inside` ou `outside`.
- Processamento em lote de múltiplos dispositivos.

O serviço escuta o tópico `nodered/wifi/data`, processa o payload e grava os registros analisados no banco.

### Backend

O módulo `backend/` contém uma API FastAPI para consulta dos dispositivos detectados, histórico, estatísticas e status do sistema.

Funcionalidades:

- API REST para dispositivos, histórico e estatísticas.
- Identificação de SO usando lookup de MAC OUI.
- Estimativa de distância por RSSI.
- Registro de histórico de detecções.
- Agregações e estatísticas.
- Criação automática das tabelas do banco.

Importante: a ingestão MQTT e a análise dos dispositivos são feitas pelo Notebook/ML. O backend consome os dados já persistidos no banco.

### Frontend

O projeto também possui um frontend em `frontend/`, preparado para consumir a API e exibir painel, lista de dispositivos, detalhes, filtros e estatísticas.

## Requisitos

- Python 3.8+
- PostgreSQL 12+
- Docker
- Docker Compose
- Node-RED e Mosquitto via `nodered/docker-compose.yml`
- MQTT CLI opcional para testes (`mosquitto_pub` e `mosquitto_sub`)

## Configuração

Configure as variáveis de ambiente do backend em `backend/.env`:

```env
DATABASE_URL=postgresql://user:password@localhost:5432/wifi_detection
```

Se o projeto for executado em módulos separados, instale as dependências Python de cada parte:

```bash
pip install -r requirements.txt
pip install -r backend/requirements.txt
pip install -r ML/requirements.txt
```

## Executando o Node-RED e o MQTT Broker

A partir da pasta `nodered/`:

```bash
docker-compose up -d
```

Para parar os serviços:

```bash
docker-compose down
```

Para acompanhar logs:

```bash
docker-compose logs -f node-red
docker-compose logs -f mosquitto
```

## Executando o Notebook/ML

A partir da raiz do projeto:

```bash
python ML/notebook.py
```

O serviço escuta o tópico `nodered/wifi/data` e salva no banco os dispositivos processados.

Tambem é possivel usar a função diretamente em Python:

```python
from ML.notebook import process_payload

payload = {
    "timestamp": "2026-05-19T10:30:45Z",
    "packets": [
        {
            "source_mac": "AA:BB:CC:DD:EE:01",
            "rssi": -55,
            "channel": 1,
            "frequency": 2412,
            "frame_type": "probe_req",
            "seen_count": 3,
        }
    ],
}

process_payload(payload)
```

## Executando o Backend

A partir da pasta `backend/`:

```bash
python -m venv venv
venv\Scripts\activate
pip install -r requirements.txt
```

Se estiver usando Alembic:

```bash
alembic upgrade head
```

Inicie a API:

```bash
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

A API ficará disponível em:

- API: `http://localhost:8000`
- Documentação Swagger: `http://localhost:8000/docs`

## Endpoints da API

### Devices

- `GET /api/devices` - Lista todos os dispositivos.
- `GET /api/devices/{mac}` - Retorna detalhes de um dispositivo.
- `POST /api/devices` - Cria ou atualiza um dispositivo.
- `GET /api/devices/{mac}/detections` - Retorna detecções de um dispositivo.

### History

- `GET /api/history/detections` - Retorna histórico de detecções.
- `GET /api/history/stats` - Retorna estatísticas.
- `GET /api/history/timeline` - Retorna dados de timeline.

### System

- `GET /` - Endpoint raiz.
- `GET /health` - Health check.

## Formatos de Dados

### Entrada do ESP32

Tópico: `esp32/wifi/scan`

```json
{
  "device_id": "esp32_001",
  "timestamp": "2026-05-19T10:30:45Z",
  "packets": [
    {
      "source_mac": "AA:BB:CC:DD:EE:FF",
      "rssi": -65,
      "channel": 1,
      "frequency": 2412,
      "frame_type": "probe_req",
      "seen_count": 3
    }
  ]
}
```

### Saída Validada do Node-RED

Tópico: `nodered/wifi/data`

```json
{
  "timestamp": "2026-05-19T10:30:45Z",
  "devices": [
    {
      "mac": "AA:BB:CC:DD:EE:FF",
      "rssi": -65,
      "channel": 1,
      "frequency": 2412,
      "frame_type": "probe_req",
      "seen_count": 3
    }
  ]
}
```

### Resultado da Análise

```json
{
  "mac_address": "AA:BB:CC:DD:EE:01",
  "so_identified": "Apple",
  "distance_estimated": 2.45,
  "location": "inside",
  "confidence": 0.75
}
```

## Banco de Dados

### `devices`

- MAC address único.
- Timestamps de primeira e última detecção.
- RSSI, canal, frequência, tipo de frame, quantidade de aparições e SSID.
- SO identificado.
- Distância estimada.

### `detections`

- MAC do dispositivo.
- Timestamp.
- RSSI.
- Canal e frequência.
- Tipo de frame e quantidade de aparições.
- Localização (`inside` ou `outside`).

### `analysis`

- MAC do dispositivo.
- SO identificado.
- Distância estimada.
- Score de confiança.
- Última atualização.

## Estimativa de Distância e Localização

A estimativa de distância usa um modelo simplificado de perda de percurso:

```text
Distance = 10^((TxPower - RSSI) / (10 * N))
```

Parâmetros usados:

- `TxPower`: `-30 dBm`, valor típico para WiFi.
- `N`: `2.0` para 5 GHz.
- `N`: `2.5` para 2.4 GHz.

Classificação de localização:

- `inside`: RSSI maior que `-70 dBm`.
- `outside`: RSSI menor ou igual a `-70 dBm`.

## Testes com MQTT CLI

Publicar uma mensagem simulada:

```bash
mosquitto_pub -h localhost -t esp32/wifi/scan -m '{
  "timestamp": "2026-05-19T10:30:45Z",
  "packets": [
    {
      "source_mac": "AA:BB:CC:DD:EE:01",
      "rssi": -55,
      "channel": 1,
      "frequency": 2412,
      "frame_type": "probe_req",
      "seen_count": 3
    }
  ]
}'
```

Assinar o tópico de saída:

```bash
mosquitto_sub -h localhost -t nodered/wifi/data
```

Monitorar tópicos:

```bash
mosquitto_sub -h localhost -v -t "nodered/#"
mosquitto_sub -h localhost -v -t "esp32/#"
```

## Troubleshooting

### MQTT connection refused

- Verifique se o container do Mosquitto está rodando: `docker ps`.
- Consulte os logs: `docker-compose logs mosquitto`.
- Verifique se a porta `1883` não está bloqueada.

### Node-RED não conecta no MQTT

- Verifique o hostname do broker. Em rede Docker, ele deve ser `mosquitto`.
- Confira a conectividade da rede Docker: `docker network ls`.
- Consulte os logs do Node-RED.

### Nenhuma mensagem trafegando

- Verifique se o ESP32 ou publicador mock está enviando para o tópico correto.
- Confira se os nomes dos tópicos estão corretos.
- Habilite os nós de debug no Node-RED.
- Use `mosquitto_sub` para confirmar se as mensagens estão sendo publicadas.

## Considerações de Produção

- Habilitar autenticação no MQTT.
- Usar TLS/SSL.
- Persistir mensagens no broker.
- Implementar backup e recovery.
- Monitorar desempenho do broker.
- Configurar logs e restart automático.
- Revisar regras de validação e tolerância a falhas.

## Melhorias Futuras

- Modelos reais de ML para identificação de SO.
- Estimativa de distância mais avançada usando múltiplas frequências.
- Classificação de tipo de dispositivo, como celular, notebook ou IoT.
- Análise de aglomeração e mapas de calor.
- Reconhecimento de padrões temporais.

## Referências

- [Node-RED Official Docs](https://nodered.org/docs/)
- [Mosquitto Documentation](https://mosquitto.org/documentation/)
- [MQTT Protocol](http://mqtt.org/)
