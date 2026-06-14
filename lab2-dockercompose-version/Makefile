.PHONY: all help dev reset build proto banner docker-build docker-VM1 docker-VM2 docker-VM3 docker-VM4 up down clean

.DEFAULT_GOAL := help

# Ayuda: lista los targets mas usados.
help:
	@echo "DiscoPass - targets disponibles:"
	@echo "  make dev          proto + build + up   (ciclo normal de desarrollo)"
	@echo "  make reset        clean + proto + build + up   (reconstruccion total)"
	@echo "  make up           docker-build + banner + levantar contenedores"
	@echo "  make down         detener todos los servicios"
	@echo "  make build        compilar binarios Go (sin Docker)"
	@echo "  make proto        regenerar codigo gRPC desde el .proto"
	@echo "  make docker-build construir las imagenes Docker"
	@echo "  make banner       imprimir el banner de arranque"
	@echo "  make clean        borrar binarios compilados"
	@echo "  make docker-VM1..4  levantar el subconjunto de servicios de cada VM"

# Ciclo de desarrollo: regenera proto, recompila y levanta todo.
dev: proto build up

# Reconstruccion total desde cero: limpia, regenera proto, recompila y levanta.
reset: clean proto build up

# Banner de arranque: comportamiento esperado, topologia y leyenda de colores.
# Se imprime despues de construir las imagenes y antes de levantar los logs.
banner:
	@./scripts/banner.sh

# Build all Go binaries (without Docker)
build:
	go build ./cmd/broker
	go build ./cmd/dbnode
	go build ./cmd/banco
	go build ./cmd/productor
	go build ./cmd/consumidor

# Regenerate gRPC code from .proto
proto:
	protoc --go_out=. --go_opt=module=discopass \
	       --go-grpc_out=. --go-grpc_opt=module=discopass \
	       proto/discopass.proto

# Build all Docker images
docker-build:
	docker-compose build

# VM1: Broker (MV1: dist057 — 10.35.168.67)
docker-VM1: banner
	docker-compose -f docker-compose-vm1.yml up

# VM2: Productores + DB3 (MV2: dist058 — 10.35.168.68)
docker-VM2: banner
	docker-compose -f docker-compose-vm2.yml up

# VM3: Consumidores + DB2 (MV3: dist059 — 10.35.168.69)
docker-VM3: banner
	docker-compose -f docker-compose-vm3.yml up

# VM4: Banco + DB1 (MV4: dist060 — 10.35.168.70)
docker-VM4: banner
	docker-compose -f docker-compose-vm4.yml up

# All services in one machine (for testing).
# Orden: 1) construir imagenes (refleja cambios de codigo), 2) banner de
# arranque, 3) levantar contenedores. Asi el banner queda DESPUES de Docker y
# ANTES de los logs del programa.
up: docker-build banner
	docker-compose up

# Stop all services
down:
	docker-compose down

# Clean build artifacts
clean:
	rm -f broker dbnode banco productor consumidor
	rm -rf *.exe
