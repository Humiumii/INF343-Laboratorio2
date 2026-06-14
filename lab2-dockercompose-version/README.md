# INF343 — Laboratorio 2: DiscoPass

**Grupo**: XX

**Integrantes**:
- Nombre Apellido — Rol 20.xxx.xxx-x
- Nombre Apellido — Rol 20.xxx.xxx-x
- Nombre Apellido — Rol 20.xxx.xxx-x

---

Sistema distribuido de venta y validación de entradas para discotecas.
Comunicación estricta por **gRPC + Protocol Buffers**.
Cada entidad corre en un contenedor Docker separado.

## Topología (4 VMs)

| VM | IP | Componentes | Puertos |
|----|----|-------------|---------|
| **dist057** | 10.35.168.67 | Broker | 50051 |
| **dist058** | 10.35.168.68 | DataClub, Dockers Night, GoLounge, GeorgieHouse + DB3 | 50063 |
| **dist059** | 10.35.168.69 | ClienteA, ClienteB + DB2 | 50062 |
| **dist060** | 10.35.168.70 | Banco USM + DB1 | 50052, 50061 |

```
Productores ──gRPC──> Broker ──gRPC──> Nodos DB (DB1, DB2, DB3)
Consumidores ──gRPC──> Broker ──gRPC──> Banco USM
```

- **Broker** = único punto de coordinación (registro, validación, idempotencia, escritura N=3/W=2, lectura R=2, resincronización de nodos caídos, reporte final).
- **Nodos DB**: replicación tipo DynamoDB con consistencia eventual. Fallo simulado interno (caída temporal + resincronización vía `SolicitarBacklog`).
- **Banco USM**: aprueba pagos con 80% probabilidad (90% si medio_pago=credito). Timeout de 3s en el broker → compra queda "pendiente" si el banco no responde.
- **Resincronización mediada por broker**: cuando un nodo revive, pide el backlog al broker (no replica nodo-a-nodo), respetando la regla de que el broker es el único punto autorizado.

## Requisitos

- Docker Engine + docker-compose (v1 o v2)
- Las 4 VMs deben tener conectividad de red entre sí (IPs 10.35.168.67-70)

## Cómo ejecutar en 4 VMs

Arrancar **dist057 primero** y esperar ~5s antes de las demás.

**VM1 — dist057 (Broker):**
```bash
cd ~/lab2/INF343-Laboratorio2/lab2-dockercompose-version
make docker-VM1
```

**VM2 — dist058 (Productores + DB3):**
```bash
cd ~/lab2/INF343-Laboratorio2/lab2-dockercompose-version
make docker-VM2
```

**VM3 — dist059 (Consumidores + DB2):**
```bash
cd ~/lab2/INF343-Laboratorio2/lab2-dockercompose-version
make docker-VM3
```

**VM4 — dist060 (Banco + DB1):**
```bash
cd ~/lab2/INF343-Laboratorio2/lab2-dockercompose-version
make docker-VM4
```

Para detener: `Ctrl+C` en cada terminal. O desde cualquier VM:
```bash
docker-compose -f docker-compose-vm1.yml down   # según la VM
```

### Opción alternativa: Todo en una máquina (pruebas)

```bash
make up
```

### Opción sin Docker (desarrollo rápido)

```bash
# Terminal 1
go run ./cmd/broker -puerto 50051 -nodos localhost:50061,localhost:50062,localhost:50063 -banco localhost:50052

# Terminales 2-4
go run ./cmd/dbnode -id DB1 -puerto 50061 -broker localhost:50051
go run ./cmd/dbnode -id DB2 -puerto 50062 -broker localhost:50051
go run ./cmd/dbnode -id DB3 -puerto 50063 -broker localhost:50051

# Terminal 5
go run ./cmd/banco -puerto 50052 -broker localhost:50051

# Terminal 6
go run ./cmd/productor -discoteca DataClub -broker localhost:50051 -catalogo config/catalogo_30.json

# Terminal 7
go run ./cmd/consumidor -id ClienteA -broker localhost:50051 -medio credito -intervalo 10
```

## Resultados

- **Reporte.txt** — se genera en el broker (dist057) cada 20s. Revisar con:
  ```bash
  ssh dist057 cat ~/lab2/INF343-Laboratorio2/lab2-dockercompose-version/Reporte.txt
  ```
- **CSV de usuarios** — se generan en dist059:
  ```bash
  ssh dist059 cat ~/lab2/INF343-Laboratorio2/lab2-dockercompose-version/usuario_ClienteA.csv
  ssh dist059 cat ~/lab2/INF343-Laboratorio2/lab2-dockercompose-version/usuario_ClienteB.csv
  ```

## Configuración (flags / variables de entorno)

| Flag | Variable | Default | Descripción |
|------|----------|---------|-------------|
| `-puerto` | `PUERTO` | `50051` (broker) | Puerto del servicio |
| `-nodos` | `NODOS` | `localhost:...` | Direcciones de nodos DB (broker) |
| `-banco` | `BANCO` | `localhost:50052` | Dirección del banco (broker) |
| `-broker` | `BROKER` | `localhost:50051` | Dirección del broker |
| `-id` | `ID` | `DB1` / `ClienteA` | Identificador de la entidad |
| `-min` / `-max` | `MIN_INTERVAL` / `MAX_INTERVAL` | `30` / `40` | Intervalo entre publicaciones (productor) |
| `-intervalo` | `INTERVALO` | `10` | Segundos entre compras (consumidor) |
| `-medio` | `MEDIO_PAGO` | `debito` | Medio de pago del consumidor |
| `-catalogo` | `CATALOGO` | `config/catalogo_30.json` | Catálogo de eventos (productor) |
| `-discoteca` | `DISCOTECA` | `DataClub` | Nombre de la discoteca (productor) |
| `-fallos` | `FALLOS` | `true` | Simular fallos temporales |

## Reportes

- **Reporte.txt**: se genera automáticamente en el directorio del broker
  (cada 20s y al finalizar). Contiene:
  1. Resumen de discotecas
  2. Estado de nodos DB
  3. Compras y tickets
  4. Estado del servicio de pago
  5. Fallos y recuperaciones
  6. Conclusión

- **CSV de usuarios**: se generan `usuario_ClienteA.csv` y `usuario_ClienteB.csv`
  con el historial de compras de cada consumidor.
