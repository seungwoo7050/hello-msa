# 키-값 스토어 및 캐시 서비스 (Redis 기반) - `redis_cache_service`

MSA (마이크로서비스 아키텍처) 환경에서 여러 애플리케이션 서비스가 공유하여 사용할 수 있도록 설계된 Redis 기반 키-값 저장소 및 캐시 서비스입니다.

## 시작하기 전에

이 `redis_cache_service`를 로컬 환경에서 빌드하고 테스트하려면 다음 소프트웨어가 시스템에 설치되어 있어야 합니다:

* **Docker**: 컨테이너 실행 환경을 제공합니다. ([Docker 공식 웹사이트](https://www.docker.com/)에서 설치)
* **Docker Compose**: 여러 컨테이너를 정의하고 실행하는 도구입니다. (Docker Desktop 설치 시 함께 제공되는 경우가 많습니다.)
* **`make`**: `Makefile`에 정의된 명령어를 실행하기 위한 도구입니다. (Linux 및 macOS에는 기본적으로 포함되어 있으며, Windows 사용자는 별도 설치가 필요할 수 있습니다. 예: Chocolatey를 통해 `choco install make` 또는 WSL 사용)

## 목차

1.  [Redis 소개 및 기본 사용법](#redis-소개-및-기본-사용법)
    * [Redis란?](#redis란)
    * [주요 데이터 타입](#주요-데이터-타입)
    * [기본 명령어 예시](#기본-명령어-예시)
    * [`redis.conf` 파일 수정 방법](#redisconf-파일-수정-방법)
2.  [서비스 실행 및 접속 정보](#서비스-실행-및-접속-정보)
    * [MSA 전체 환경에서 실행 시](#msa-전체-환경에서-실행-시)
    * [로컬 테스트 환경에서 실행 시 (`Makefile` 사용)](#로컬-테스트-환경에서-실행-시-makefile-사용)
3.  [다른 서비스에서 Redis 사용 방법](#다른-서비스에서-redis-사용-방법)
    * [Python 예시 (`redis-py`)](#python-예시-redis-py)
    * [Node.js 예시 (`ioredis`)](#nodejs-예시-ioredis)
    * [Java 예시 (`Jedis`)](#java-예시-jedis)
4.  [테스트 가이드 (`Makefile` 사용)](#테스트-가이드-makefile-사용)
    * [테스트 목적](#테스트-목적)
    * [주요 `make` 명령어](#주요-make-명령어)
    * [테스트 항목 상세](#테스트-항목-상세)
5.  [디렉터리 구조](#디렉터리-구조)

---

## Redis 소개 및 기본 사용법

### Redis란?

Redis (Remote Dictionary Server)는 오픈 소스 기반의 **인메모리 데이터 구조 저장소**입니다. 주로 데이터베이스, 캐시, 메시지 브로커 등으로 사용됩니다. 키-값(Key-Value) 저장소 모델을 따르며, 다양한 종류의 데이터 구조(문자열, 해시, 리스트, 세트, 정렬된 세트 등)를 지원하여 빠른 속도와 높은 유연성을 제공합니다.

* **버전 정보**: 이 서비스는 `redis:8.0-alpine` 이미지를 기반으로 구성됩니다.
* **주요 사용 사례**:
    * **캐싱**: 자주 접근하는 데이터나 API 응답 결과를 임시 저장하여 DB 부하 감소 및 응답 속도 향상.
    * **세션 관리**: 웹 애플리케이션 사용자 세션 정보 저장.
    * **실시간 순위표**: 게임이나 서비스의 실시간 랭킹 구현.
    * **메시징 큐**: 간단한 메시지 발행/구독(Pub/Sub) 또는 작업 큐 구현.

### 주요 데이터 타입

* **Strings**: 가장 기본적인 타입으로, 문자열, 숫자, 바이너리 데이터를 저장할 수 있습니다 (최대 512MB).
* **Hashes**: 필드-값 쌍으로 구성된 객체를 저장하기에 적합합니다 (예: 사용자 프로필).
* **Lists**: 문자열 요소들의 순서 있는 컬렉션입니다. 큐(Queue)나 스택(Stack)으로 활용하기 좋습니다.
* **Sets**: 순서 없는 고유한 문자열 요소들의 컬렉션입니다. 중복 제거, 교집합/합집합 등의 연산에 유용합니다.
* **Sorted Sets (ZSETs)**: 각 문자열 요소가 '점수(score)'와 연결된 정렬된 세트입니다. 점수를 기준으로 정렬되므로 순위표 구현에 매우 효과적입니다.

### 기본 명령어 예시

`redis-cli`를 통해 Redis 서버에 직접 명령을 실행할 수 있습니다.

* `PING` : 서버가 정상 동작하는지 확인 (응답: `PONG`).
* `SET mykey "Hello"` : `mykey`라는 키에 "Hello"라는 값을 저장.
* `GET mykey` : `mykey`의 값을 조회 (응답: `"Hello"`).
* `EXPIRE mykey 60` : `mykey`에 60초의 만료 시간 설정.
* `TTL mykey` : `mykey`의 남은 만료 시간 확인.
* `DEL mykey` : `mykey` 삭제.
* `HSET user:1000 name "Alice" age 30` : `user:1000` 해시에 `name`과 `age` 필드 설정.
* `HGETALL user:1000` : `user:1000` 해시의 모든 필드와 값 조회.
* `LPUSH mylist "world"` `LPUSH mylist "hello"` : `mylist` 리스트의 왼쪽에 "hello", "world" 순으로 값 추가.
* `LRANGE mylist 0 -1` : `mylist`의 모든 요소 조회.

### `redis.conf` 파일 수정 방법

Redis 서버의 상세 설정은 `config/redis.conf` 파일에서 관리합니다. (한국어 사용자를 위한 주요 설정 가이드)

* **파일 위치**: `services/redis_cache_service/config/redis.conf`
* **주요 설정 항목**:
    * `bind 127.0.0.1 -::1`: 특정 IP 주소에서만 접속을 허용합니다. Docker 환경에서는 보통 컨테이너 내부에서 모든 IP를 허용하도록 `bind 0.0.0.0` 또는 이 줄을 주석 처리합니다. (Docker 환경에서 다른 서비스나 호스트로부터의 안정적인 접속을 위해서는 `config/redis.conf` 파일에 `bind 0.0.0.0` 설정을 명시하는 것을 권장합니다. `requirepass`를 통해 비밀번호 인증을 사용하므로 이와 같이 설정해도 보안이 유지됩니다. 만약 `bind` 설정이 `127.0.0.1`로 되어 있거나 주석 처리되어 기본값을 따를 경우, 컨테이너 외부에서의 접속이 제한될 수 있습니다.)
    * `port 6379`: Redis가 리스닝할 포트 번호입니다.
    * `requirepass yourSuperSecurePassword123!`: Redis 접속 시 필요한 비밀번호를 설정합니다. **반드시 강력한 비밀번호로 변경하세요.**
    * `appendonly yes`: AOF(Append Only File) 영속성 방식을 사용합니다. 모든 쓰기 작업을 로그 파일에 기록하여 데이터 손실 위험을 줄입니다.
    * `save 900 1`: RDB(Snapshot) 영속성 방식 설정 예시입니다. 900초(15분) 동안 1개 이상의 키 변경이 있으면 스냅샷을 저장합니다. AOF와 함께 사용 가능합니다.
    * `maxmemory <bytes>`: Redis가 사용할 최대 메모리 양을 설정합니다. (예: `maxmemory 256mb`)
    * `maxmemory-policy allkeys-lru`: `maxmemory`에 도달했을 때, 가장 최근에 사용되지 않은 키(LRU)부터 제거하는 정책입니다. 캐시 용도로 많이 사용됩니다.
* **수정 후 적용**:
    * `config/redis.conf` 파일을 수정한 후에는 Redis 서비스를 재시작해야 변경 사항이 적용됩니다.
    * **로컬 테스트 환경 (`Makefile` 사용 시)**: `make down && make up` 명령으로 서비스를 재시작합니다. (`make up` 시 `--build` 옵션이 있다면 이미지를 다시 빌드합니다.)
    * **MSA 전체 환경 (루트 `docker-compose.yml` 사용 시)**: 일반적으로 `docker compose down && docker compose up -d --build <service_name>` 과 같이 해당 서비스를 재빌드/재시작합니다.

---

## 서비스 실행 및 접속 정보

### MSA 전체 환경에서 실행 시

MSA 프로젝트의 루트 `docker-compose.yml` 파일에 이 `redis_cache_service`가 포함되어 빌드되고 실행될 때, 다른 서비스에서 이 Redis에 접속하는 정보는 다음과 같습니다.

* **호스트명**: `redis_cache` (루트 `docker-compose.yml`에 정의된 Redis 서비스의 이름)
* **포트**: `6379` (Redis 컨테이너 내부 기본 포트)
* **비밀번호**: `config/redis.conf` 파일의 `requirepass` 지시어에 설정된 값 (예: `yourSuperSecurePassword123!`)
* **네트워크**: 다른 서비스와 동일한 Docker 네트워크에 연결되어 있어야 합니다.

```yaml
# MSA 프로젝트의 루트 docker-compose.yml 파일에 추가될 내용입니다.
# version: '3.8' # 파일 상단에 이미 버전이 명시되어 있다면 생략 가능

services:
  # --- Redis Cache Service 시작 ---
  redis_cache:
    # 'services/redis_cache_service/' 디렉터리에 있는 Dockerfile을 사용하여 이미지를 빌드합니다.
    # 해당 디렉터리에는 Redis 설정을 포함한 Dockerfile이 있어야 합니다.
    build:
      context: ./services/redis_cache_service
      dockerfile: Dockerfile # Dockerfile 이름이 다른 경우 명시적으로 지정
    container_name: msa_redis_cache # 컨테이너에 고유한 이름 부여
    restart: always # 컨테이너 비정상 종료 시 항상 재시작
    ports:
      # 호스트의 6379 포트를 컨테이너의 6379 포트로 매핑합니다.
      # 개발/디버깅 목적으로 호스트에서 직접 Redis에 접근할 때 사용합니다.
      # 프로덕션 환경에서는 보안을 위해 이 포트 노출을 제한할 수 있습니다.
      - "6379:6379"
    volumes:
      # Redis 데이터를 영속적으로 저장하기 위한 명명된 볼륨입니다.
      # 컨테이너가 재시작되어도 데이터가 유지됩니다.
      # 'redis.conf' 파일은 이미지 빌드 시 포함되므로 여기서 마운트하지 않습니다.
      - redis_data:/data
    networks:
      # MSA 내 다른 서비스와 통신하기 위한 공용 네트워크입니다.
      # 이 네트워크는 docker-compose.yml 파일 하단에 정의되어야 합니다.
      - msa_internal_network # 예시 네트워크 이름, 실제 사용하는 네트워크 이름으로 변경
  # --- Redis Cache Service 종료 ---

  # ... 여기에 다른 마이크로서비스들을 정의합니다.
  # 예시:
  # product_service:
  #   build: ./services/product_service
  #   ports:
  #     - "8001:8000"
  #   networks:
  #     - msa_internal_network
  #   depends_on:
  #     - redis_cache

# Docker Compose 파일 하단 (또는 상단)에 다음을 정의합니다.
volumes:
  redis_data: # Redis 데이터 저장을 위한 명명된 볼륨 선언
  # ... 다른 서비스들이 사용할 수 있는 추가 볼륨들 ...

networks:
  msa_internal_network: # 모든 서비스가 공유할 브릿지 네트워크 선언
    driver: bridge
  # ... 필요에 따라 추가 네트워크 정의 ...
```

### 로컬 테스트 환경에서 실행 시 (`Makefile` 사용)

이 `redis_cache_service` 폴더 내에서 `Makefile`을 사용하여 독립적으로 테스트할 때의 접속 정보입니다. (기준: `docker-compose.test.yaml`)

* **실행 명령어**: `make up`
* **호스트명 (외부에서 접속 시)**: `localhost`
* **포트 (외부에서 접속 시)**: `16379` (호스트의 `16379` 포트가 컨테이너의 `6379` 포트로 매핑됨)
* **비밀번호**: `config/redis.conf` 파일의 `requirepass` 지시어에 설정된 값 (예: `yourSuperSecurePassword123!`)

---

## 다른 서비스에서 Redis 사용 방법

다음은 다양한 프로그래밍 언어로 작성된 클라이언트 서비스에서 이 Redis 서비스에 연결하고 사용하는 기본 예제입니다.

**공통 접속 정보 (예시):**

* MSA 내부 호스트: `redis_cache`
* 로컬 테스트 외부 호스트: `localhost`
* MSA 내부 포트: `6379`
* 로컬 테스트 외부 포트: `16379`
* 비밀번호: `yourSuperSecurePassword123!` (실제 `redis.conf`에 설정된 값 사용)
* 사용할 DB 번호: (예: `0`)

### Python 예시 (`redis-py`)

* **라이브러리 설치**: `pip install redis`
* **코드 예시 (`main.py`)**:

```python
import redis

try:
    # 로컬 테스트 환경 접속 정보 (외부에서 접속 시)
    # r = redis.Redis(host='localhost', port=16379, db=0, password='yourSuperSecurePassword123!', decode_responses=True)

    # MSA 내부 다른 서비스에서 접속 시
    r = redis.Redis(host='redis_cache', port=6379, db=0, password='yourSuperSecurePassword123!', decode_responses=True)
    
    # 연결 확인
    r.ping()
    print("Python: Successfully connected to Redis!")

    # 데이터 설정
    r.set('python_message', 'Hello from Python!')
    r.expire('python_message', 60) # 60초 후 만료

    # 데이터 조회
    message = r.get('python_message')
    print(f"Python: Message from Redis: {message}")

except redis.exceptions.ConnectionError as e:
    print(f"Python: Could not connect to Redis: {e}")
except redis.exceptions.ResponseError as e:
    print(f"Python: Redis command failed (check password or command syntax): {e}")

# 비동기 처리가 필요하다면 `aioredis` 라이브러리 사용을 고려하세요.
# import aioredis
# async def main_async():
#     try:
#         redis_async = await aioredis.from_url(f"redis://:{'yourSuperSecurePassword123!'}@redis_cache:6379/0", decode_responses=True)
#         await redis_async.ping()
#         print("Python (aioredis): Successfully connected to Redis!")
#         await redis_async.set('python_async_message', 'Hello from Aioredis!')
#         message_async = await redis_async.get('python_async_message')
#         print(f"Python (aioredis): Message from Redis: {message_async}")
#         await redis_async.close()
#     except Exception as e:
#         print(f"Python (aioredis): Could not connect or run command: {e}")
# if __name__ == '__main__':
#     # asyncio.run(main_async())
#     pass
```

### Node.js 예시 (`ioredis`)

* **라이브러리 설치**: `npm install ioredis`
* **코드 예시 (`app.js`)**:

```javascript
const IORedis = require('ioredis');

// 로컬 테스트 환경 접속 정보 (외부에서 접속 시)
// const redis = new IORedis({
//   host: 'localhost',
//   port: 16379,
//   password: 'yourSuperSecurePassword123!',
//   db: 0,
// });

// MSA 내부 다른 서비스에서 접속 시
const redis = new IORedis({
  host: 'redis_cache', // Docker 서비스 이름
  port: 6379,
  password: 'yourSuperSecurePassword123!',
  db: 0, // 사용할 데이터베이스 번호
});

redis.on('connect', () => {
  console.log('Node.js: Successfully connected to Redis!');
});

redis.on('error', (err) => {
  console.error('Node.js: Could not connect to Redis:', err);
});

async function runRedisExample() {
  try {
    await redis.set('node_message', 'Hello from Node.js!', 'EX', 60); // EX 60: 60초 후 만료
    const message = await redis.get('node_message');
    console.log(`Node.js: Message from Redis: ${message}`);
  } catch (error) {
    console.error('Node.js: Redis command failed:', error);
  } finally {
    // 필요에 따라 연결을 명시적으로 닫을 수 있지만, 보통 애플리케이션 생명주기 동안 유지합니다.
    // redis.quit(); 
  }
}

// 연결이 establecida 될 때까지 잠시 기다린 후 예제 실행
setTimeout(runRedisExample, 1000); // 실제 사용 시에는 'connect' 이벤트를 활용하는 것이 더 좋습니다.
```

### Java 예시 (`Jedis`)

* **라이브러리 의존성 추가 (Maven `pom.xml`)**:

```xml
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>5.1.3</version> </dependency>
```

* **코드 예시 (`RedisJavaExample.java`)**:

```java
import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPool;
import redis.clients.jedis.JedisPoolConfig;
import redis.clients.jedis.exceptions.JedisConnectionException;
import redis.clients.jedis.exceptions.JedisDataException;

public class RedisJavaExample {
    public static void main(String[] args) {
        // JedisPool을 사용하여 커넥션 관리 (권장)
        JedisPoolConfig poolConfig = new JedisPoolConfig();
        // 로컬 테스트 환경 접속 정보 (외부에서 접속 시)
        // JedisPool jedisPool = new JedisPool(poolConfig, "localhost", 16379, 2000, "yourSuperSecurePassword123!", 0);
        
        // MSA 내부 다른 서비스에서 접속 시
        JedisPool jedisPool = new JedisPool(poolConfig, "redis_cache", 6379, 2000, "yourSuperSecurePassword123!", 0);

        try (Jedis jedis = jedisPool.getResource()) {
            // 연결 확인
            if ("PONG".equals(jedis.ping())) {
                System.out.println("Java: Successfully connected to Redis!");
            }

            // 데이터 설정 (60초 후 만료)
            jedis.setex("java_message", 60, "Hello from Java!");

            // 데이터 조회
            String message = jedis.get("java_message");
            System.out.println("Java: Message from Redis: " + message);

        } catch (JedisConnectionException e) {
            System.err.println("Java: Could not connect to Redis: " + e.getMessage());
            // e.printStackTrace();
        } catch (JedisDataException e) {
            System.err.println("Java: Redis command failed (check password or command syntax): " + e.getMessage());
        } finally {
            if (jedisPool != null) {
                jedisPool.close();
            }
        }
    }
}
```

**참고**: 실제 프로덕션 환경에서는 Connection Pooling, 오류 처리, 재시도 로직 등을 견고하게 구현해야 합니다.

---

## 테스트 가이드 (`Makefile` 사용)

이 서비스의 기본적인 기능과 설정을 검증하기 위해 `Makefile`을 사용한 테스트 자동화 스크립트가 제공됩니다.

### 테스트 목적

* Redis 서비스 컨테이너가 정상적으로 빌드되고 실행되는지 확인합니다.
* 설정된 비밀번호로 인증이 성공하는지 확인합니다.
* 기본적인 데이터 저장(SET) 및 조회(GET) 작업이 올바르게 동작하는지 확인합니다.
* `redis.conf`에 정의된 주요 설정(예: `appendonly`, `maxmemory`)이 적용되었는지 확인합니다.
* 데이터 영속성(AOF/RDB)이 올바르게 동작하여 서비스 재시작 후에도 데이터가 유지되는지 확인합니다.

### 주요 make 명령어

`services/redis_cache_service/` 디렉터리에서 다음 명령어를 실행할 수 있습니다.

* `make help`: 사용 가능한 모든 `make` 명령어와 설명을 보여줍니다.
* `make up`: 테스트용 Redis 서비스를 시작합니다. (이미지 빌드 포함)
* `make test_ping`: Redis 서버에 PING을 보내 응답을 확인하고 인증을 테스트합니다.
* `make test_set_get`: 간단한 키-값 설정 및 조회 작업을 테스트합니다.
* `make test_config`: `redis.conf`의 주요 설정이 올바르게 적용되었는지 확인합니다.
* `make test_persistence`: 데이터를 저장 후 서비스 재시작 시 데이터가 유지되는지 확인합니다.
* `make test_all`: 위의 `test_ping`, `test_set_get`, `test_config` 테스트를 순차적으로 실행하고, 성공 시 테스트 환경을 정리합니다.
* `make down`: 테스트용 Redis 서비스를 중지하고 관련 컨테이너, 이 서비스용으로 생성된 네트워크, 그리고 명명된 볼륨을 삭제합니다. 하지만, `make up` 또는 `docker compose build` 명령으로 로컬에 빌드된 Docker 이미지는 삭제하지 않습니다. 빌드된 이미지까지 모두 정리하려면 `make clean` 명령을 사용하세요.
* `make clean`: `down` 명령어의 기능에 더해, 로컬에서 빌드된 서비스 이미지까지 삭제합니다 (`--rmi local` 옵션 사용).
* `make logs`: 실행 중인 Redis 컨테이너의 로그를 실시간으로 확인합니다.
* `make cli`: 실행 중인 Redis 컨테이너의 `redis-cli`에 직접 접속합니다 (비밀번호 입력 필요).

### 테스트 항목 상세

* `test_ping`: `redis-cli PING` 명령을 실행하여 "PONG" 응답을 확인합니다. `redis.conf`에 설정된 비밀번호를 사용하여 인증을 시도합니다.
* `test_set_get`: 임의의 키-값을 `SET` 명령으로 Redis에 저장한 후, `GET` 명령으로 해당 값을 다시 읽어와 일치하는지 비교합니다. 테스트 후 사용된 키는 `DEL` 명령으로 삭제합니다.
* `test_config`: `redis-cli CONFIG GET` 명령을 사용하여 `appendonly`, `maxmemory` 등의 설정 값이 `redis.conf`에 지정된 대로 올바르게 적용되었는지 확인합니다.
* `test_persistence`: 임의의 키-값을 저장한 후 Redis 서비스를 중지했다가 다시 시작합니다. 이후 해당 키의 값이 여전히 존재하는지 확인하여 데이터 영속성(AOF 또는 RDB)이 정상 작동하는지 검증합니다. 테스트 후 사용된 키는 삭제하고 환경을 정리합니다.

---

## 디렉터리 구조

```sh
services/
└── redis_cache_service/
    ├── config/
    │   └── redis.conf        # Redis 서버 설정 파일
    ├── Dockerfile            # Redis 서비스 이미지를 빌드하기 위한 Dockerfile
    ├── docker-compose.test.yaml # 로컬 테스트 및 서비스 정의 참조용 Docker Compose 파일
    ├── Makefile              # 빌드, 테스트, 정리 작업을 자동화하는 Makefile
    └── README.md             # 현재 이 파일
```