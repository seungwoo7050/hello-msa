# Makefile for testing the Redis Cache Service

# Docker Compose
COMPOSE_FILE := docker-compose.test.yaml
COMPOSE_CMD := docker compose -f $(COMPOSE_FILE)

# Redis Contaier info
REDIS_CONTAINER_NAME := redis_service_test_container
REDIS_PORT := 16379
REDIS_PASSWORD := SecurePassword123!

# redis-cli command
EXEC_REDIS_CLI = docker exec $(REDIS_CONTAINER_NAME) redis-cli -a $(REDIS_PASSWORD)

.PHONY: help
help:
	@echo "Makefile for Redis Cache Service Testing"
	@echo ""
	@echo "Usage:"
	@echo "  make up             - Start the Redis service for testing"
	@echo "  make down           - Stop and remove the Redis service and volumes"
	@echo "  make logs           - View logs of the Redis service"
	@echo "  make status         - Show status of the service containers"
	@echo "  make cli            - Access redis-cli inside the container"
	@echo "  make test_ping      - Test PING to Redis server"
	@echo "  make test_auth      - Test authentication (implicitly done by test_ping with password)"
	@echo "  make test_set_get   - Test basic SET and GET operations"
	@echo "  make test_config    - Check some key Redis configurations"
	@echo "  make test_persistence - Test data persistence after restart (if configured)"
	@echo "  make test_all       - Run all basic tests (ping, set/get, config)"
	@echo "  make clean          - Alias for 'down'"
	@echo ""

.PHONY: up
up:
	@echo "Starting Redis service for testing..."
	$(COMPOSE_CMD) up -d --remove-orphans
	@echo "Waiting for Redis to start..."
	@sleep 5

.PHONY: down
down:
	@echo "Stopping and removing Redis service..."
	$(COMPOSE_CMD) down -v --remove-orphans --rmi local

.PHONY: logs
logs:
	@echo "Viewing logs of the Redis service..."
	$(COMPOSE_CMD) logs -f redis_under_test

.PHONY: status
status:
	@echo "Showing status of the service containers..."
	$(COMPOSE_CMD) ps

.PHONY: cli
cli: up
	@echo "Connecting to redis-cli (type 'exit' or Ctrl+D to quit)..."
	@echo "Password: $(REDIS_PASSWORD)"
	docker exec -it $(REDIS_CONTAINER_NAME) redis-cli -a $(REDIS_PASSWORD)

.PHONY: test_ping
test_ping: up
	@echo "Testing PING to Redis server..."
	@if $(EXEC_REDIS_CLI) ping | grep -q "PONG"; then \
		echo "PING test passed!"; \
	else \
		echo "PING test failed!"; \
		exit 1; \
	fi

# Authentication test (indirectly tested in test_ping)
.PHONY: test_auth
test_auth: test_ping
	@echo "Authentication test: PASSED (if PING test passed with password)"

# Basic SET and GET test
.PHONY: test_config
test_config: up
	@echo "Checking Redis configurations..."
	@echo "    Checking 'requirepass' (should be yes if password is set)..."
	@if ! docker exec $(REDIS_CONTAINER_NAME) redis-cli PING 2>/dev/null | grep -q PONG; then \
		echo "  'requirepass' seems active (connection without password failed as expected): PASSED"; \
	else \
		echo "  'requirepass' test: FAILED (connection without password succeeded or other error)"; \
	fi

	@echo "  Checking 'appendonly' (should be 'yes' from redis.conf)..."
	@AOF_STATUS=$$($(EXEC_REDIS_CLI) CONFIG GET appendonly | sed -n '2p'); \
	if [ "$$AOF_STATUS" = "yes" ]; then \
		echo "  'appendonly' config: PASSED (is 'yes')"; \
	else \
		echo "  'appendonly' config: FAILED (Expected 'yes', Got '$$AOF_STATUS')"; \
		# exit 1; \ # 필요에 따라 실패 시 중단
	fi

	@echo "  Checking 'maxmemory' (should be '268435456' for 256mb)..." # 256 * 1024 * 1024
	@MAXMEMORY_VALUE=$$($(EXEC_REDIS_CLI) CONFIG GET maxmemory | sed -n '2p'); \
	EXPECTED_MAXMEMORY="268435456"; \
	if [ "$$MAXMEMORY_VALUE" = "$(EXPECTED_MAXMEMORY)" ]; then \
		echo "  'maxmemory' config: PASSED (is '$(EXPECTED_MAXMEMORY)')"; \
	else \
		echo "  'maxmemory' config: FAILED (Expected '$(EXPECTED_MAXMEMORY)', Got '$$MAXMEMORY_VALUE')"; \
		# exit 1; \
	fi

# data persistence test
.PHONY: test_persistence
	@echo "Testing data persistence..."
	@echo "  Step 1: Start Redis and set a key."
	@make up SILENT=true
	@PERSIST_KEY="persistence_test_key"
	@PERSIST_VALUE="this_should_persist"
	@$(EXEC_REDIS_CLI) SET $(PERSIST_KEY) "$(PERSIST_VALUE)" > /dev/null
	@echo "  Key '$(PERSIST_KEY)' set. Value: '$(PERSIST_VALUE)'"
	@echo "  Step 2: Stop and restart Redis service."
	$(COMPOSE_CMD) stop redis_under_test > /dev/null
	$(COMPOSE_CMD) rm -f redis_under_test > /dev/null # 컨테이너만 삭제, 볼륨 유지
	$(COMPOSE_CMD) up -d redis_under_test > /dev/null
	@echo "  Waiting for Redis to restart..."
	@sleep 5
	@echo "  Step 3: Check if the key still exists."
	@RETRIEVED_PERSIST_VALUE=$$($(EXEC_REDIS_CLI) GET $(PERSIST_KEY)); \
	if [ "$$RETRIEVED_PERSIST_VALUE" = "$(PERSIST_VALUE)" ]; then \
		echo "  Persistence test: PASSED. Key '$(PERSIST_KEY)' found with correct value." ; \
		$(EXEC_REDIS_CLI) DEL $(PERSIST_KEY) > /dev/null; \
	else \
		echo "  Persistence test: FAILED. Key '$(PERSIST_KEY)' not found or incorrect value. Got '$$RETRIEVED_PERSIST_VALUE'"; \
		$(EXEC_REDIS_CLI) DEL $(PERSIST_KEY) > /dev/null; \
		make down SILENT=true; \
		exit 1; \
	fi
	@make down SILENT=true # 테스트 후 정리


# Run all basic tests
.PHONY: test_all
test_all: test_ping test_set_get test_config
	@echo "All basic tests completed."
	@make down SILENT=true

# 정리 (down의 별칭)
.PHONY: clean
clean: down