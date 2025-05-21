# services/redis_cache_service/Dockerfile
FROM redis:7.4-alpine

# 작업 디렉터리 설정 (선택 사항)
# WORKDIR /usr/local/etc/redis

# Redis 설정 파일 복사
COPY ./config/redis.conf /usr/local/etc/redis/redis.conf

# Redis 서버 실행 (설정 파일 사용)
CMD [ "redis-server", "/usr/local/etc/redis/redis.conf" ]