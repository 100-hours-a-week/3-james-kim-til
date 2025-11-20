# Docker: Volume / Bridge Network / Reverse Proxy

---

## 1️⃣ Docker Volume

### 1.1 Volume이란?
- 컨테이너 밖(Host)에 있는, Docker가 관리하는 영구 저장소
- 컨테이너는 지웠다 다시 만드는게 기본 전제라,
  지워져도 안 날아가야 하는 데이터(DB, logs, Upload File 등)을 볼륨에 둔다.

### 1.2 Volume vs Bound mount vs tmpfs 간단 비교
| 방식         | 저장 위치                         | 특징/언제 쓰나                  |
| ---------- | ----------------------------- | ------------------------- |
| Volume     | `/var/lib/docker/volumes/...` | 도커가 관리, 배포 환경에서 **기본 선택** |
| Bind mount | 호스트의 특정 디렉터리(임의 경로)           | 개발 환경에서 코드 공유, 호스트 디렉토리 접근 |
| tmpfs      | 메모리(RAM)                      | 컨테이너 실행 중에만, 민감/임시 데이터    |

- 운영(프로덕션): 웬만하면 Volume
- 로컬 개발: 소스 코드 공유 → Bind mount
- 속도·보안↑, 휘발 OK: tmpfs

### 1.3 기본 사용 예시
1) Volume 생성 & 컨테이너에 마운트
```bash
# 볼륨 생성
docker volume create app-data

# 애플리케이션 컨테이너에 마운트
docker run -d \
  --name app \
  -v app-data:/app/data \
  my-app-image

```

2) MySQL + Volume (많이 쓰는 패턴)
```bash
docker volume create mysql-data

docker run -d \
  --name mysql \
  -e MYSQL_ROOT_PASSWORD=pass \
  -v mysql-data:/var/lib/mysql \
  mysql:8
```

### 1.4 같은 Volume을 여러 컨테이너가 공유할 때
1) 기본 개념
```bash
docker run -d --name writer -v shared:/data ubuntu
docker run -d --name reader -v shared:/data ubuntu
```
- 두 컨테이너는 둘 다 `/data`안을 똑같이 공유한다.

> 💡 Docker가 락(lock)을 자동으로 걸어주지 않는다.  
> 그냥 디렉토리에 동시에 쓰는 것과 동일하다.  

2) "마지막에 쓴 놈이 이긴다"
예를 들어:
- `writer1` 이 `/data/test.txt` 에 `AAA` 저장
- 동시에 `writer2` 가 `/data/test.txt` 에 `BBB` 저장   
→ 파일 시스템 레벨에서 동시 쓰기 경쟁이 발생하고, 마지막에 기록된 내용으로 파일이 덮어써진다.   
→ Docker가 중간에서 “이거 동시에 쓰니까 잠시 기다려” 같은 동기화 처리를 안 해줌   
→ 결국 애플리케이션 레벨에서 잠금/트랜잭션을 직접 설계해야 한다.

---

<br/>


## 2️⃣ Bridge Network
### 2.1 Bridge Network란?
- Docker가 기본으로 만들어주는 가상 스위치 같은 것
- 같은 브릿지 네트워크에 있는 컨테이너끼리는 내부 IP / 컨테이너 이름으로 바로 통신 가능
- 외부로 나갈 때는 호스트의 IP를 통해 NAT로 나간다.
- 컨테이너들끼리 외부에 노출되지 않은 내부망에서 통신

### 2.2 사용자 정의 Bridge 네트워크 생성
```bash
# 사용자 정의 브릿지 네트워크
docker network create app-net
```

- 네트워크 정보 확인:
```bash
docker network inspect app-net
```

- 내부 서브넷, 게이트웨이, 연결된 컨테이너 목록 확인 가능

### 2.3 같은 네트워크에서 컨테이너 통신 예시
상황
- `backend` 컨테이너: 8080 포트 (REST API)
- db 컨테이너: MySQL
```bash
docker network create app-net

docker run -d \
  --name db \
  --network app-net \
  -e MYSQL_ROOT_PASSWORD=pass \
  mysql:8

docker run -d \
  --name backend \
  --network app-net \
  -e DB_HOST=db \
  my-backend
```

- `backend` 컨테이너 안에서는 `DB_HOST=db`로 그냥 이름으로 통신 가능

### 2.4 서로 다른 네트워크 간 통신: "공통 Bridge" 예시
상황
- `front-net` 에는 `frontend` 컨테이너
- `back-net` 에는 `backend` 컨테이너
- 둘은 서로 다른 네트워크라 바로 통신 불가
- 중간에 두 네트워크 모두에 붙는 컨테이너(또는 reverse proxy)를 하나 둬서 연결한다.
```bash
# 두 개의 브릿지 네트워크 생성
docker network create front-net
docker network create back-net

# 프론트 컨테이너
docker run -d \
  --name frontend \
  --network front-net \
  my-frontend

# 백엔드 컨테이너
docker run -d \
  --name backend \
  --network back-net \
  my-backend
```
여기까지 하면 `frontend` ↔ `backend` 직접 통신 X   
공통 브릿지 역할 컨테이너 추가   
예: api-gateway 컨테이너를 두 네트워크에 모두 붙이기   
```bash
docker run -d \
  --name api-gateway \
  --network front-net \
  nginx:stable-alpine

# 실행 중인 컨테이너를 다른 네트워크에도 추가 연결
docker network connect back-net api-gateway
```
이제 구조는
```bash
front-net: [frontend] <--> [api-gateway]
back-net:                  [api-gateway] <--> [backend]
```
- `frontend`는 `api-gateway` 로만 통신
- `api-gateway`는 내부에서 `backend`로 프록시
- 이렇게 네트워크를 완전히 하나로 섞지 않고, 중간에 “공통 브릿지 역할” 컨테이너를 세워서 망을 계층적으로 나눌 수 있음

> 실제로는 이 api-gateway 역할을 Nginx Reverse Proxy 로 구현하는 경우가 많다.

---

<br/>

## 3️⃣ Reverse Proxy (Nginx)
### 3.1 개념
- 클라이언트가 직접 애플리케이션 서버에 붙지 않고, 앞단에 있는 프록시 서버(Nginx 등)에 먼저 붙는다.
- Reverse Proxy가 요청을 적절한 백엔드로 전달하고, 응답을 다시 클라이언트에게 되돌려주는 중개자 역할

### 3.2 왜 쓸까?
- 하나의 엔드포인트로 여러 서비스 묶기
  - `/api` → 백엔드
  - `/` → 프론트 정적 파일
- 보안
  - 내부 서버 IP/포트 노출 최소화
- 로드 밸런싱
  - 여러 백엔드 인스턴스로 트래픽 분산
- SSL 종료(SSL Termination)
  - HTTPS → Nginx에서 복호화 → 내부는 HTTP로 통신

### 3-3. 간단한 예시 (설정 파일 nginx.conf)
```nginx
server {
    listen 80;

    location / {
        proxy_pass http://express-app:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

```
- 핵심은 proxy_pass http://express-app:8080;
  - 컨테이너 이름으로 라우팅한다는 점

### 3-4. Forward Proxy와 한 줄 비교
| 종류            | 위치          | 설명                                 |
| ------------- | ----------- | ---------------------------------- |
| Forward Proxy | **클라이언트 앞** | 클라이언트가 외부 인터넷에 나갈 때 대신 요청 (우회/필터링) |
| Reverse Proxy | **서버 앞**    | 클라이언트 요청을 받아 내부 서버로 라우팅 (보안/로드밸런싱) |

---

<br/>

## 4️⃣ Volume + Bridge Network + Reverse Proxy 종합 예시
상황
- 한 EC2 안에서:
  - `backend` (Spring/Express 등)
  - `db` (MySQL)
  - `nginx` (Reverse Proxy)
- 데이터는 Volume으로 보존
- 통신은 Bridge Network로 격리
- 외부 진입점은 Nginx 하나
```bash
# 1) 네트워크 & 볼륨
docker network create app-net
docker volume create mysql-data
docker volume create uploads

# 2) DB
docker run -d \
  --name db \
  --network app-net \
  -v mysql-data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=pass \
  mysql:8

# 3) 백엔드
docker run -d \
  --name backend \
  --network app-net \
  -v uploads:/app/uploads \
  -e DB_HOST=db \
  my-backend-image

# 4) Nginx (Reverse Proxy)
# nginx.conf는 backend:8080 으로 proxy_pass 설정
docker run -d \
  --name nginx \
  --network app-net \
  -p 80:80 \
  -v $(pwd)/nginx.conf:/etc/nginx/conf.d/default.conf \
  nginx:stable-alpine
```
- Volume
  - `mysql-data` : DB 영속성
  - `uploads` : 업로드된 파일(이미지 등)
- Bridge Network (app-net)
  - nginx, backend, db 서로 내부 IP + 컨테이너 이름으로 통신
- Reverse Proxy
  - 클라이언트는 오직 Nginx IP/도메인 + 포트만 알면 됨
  - 내부 구조(DB, 포트, 서브넷 등)는 외부에 노출되지 않음
