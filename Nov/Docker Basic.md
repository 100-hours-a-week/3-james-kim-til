# TIL: Docker 기초

## 1️⃣ Docker란?
- 애플리케이션 실행 환경(코드, 라이브러리, 설정)을 이미지로 패키징해
  어디서든 동일하게 실행되는 컨테이너 기술
- VM보다 가볍고 빠름 (커널 공유 + 격리된 프로세스 방식)

---

## 2️⃣ Docker 구성 요소 핵심
### ✔️ Image
- 실행에 필요한 모든 것을 포함한 읽기 전용 템플릿
- 여러 개의 레이어로 구성 (캐시 활용 ➡️ 빠른 빌드)

  
### ✔️ Container
- 이미지를 실제로 실행한 격리된 프로세스
- 컨테이너 내부에 저장한 변경사항은 보통 휘발성임
  ➡️ 영속화는 `Volume` 또는 바인드 마운트 필요

<img width="600" height="400" alt="image" src="https://github.com/user-attachments/assets/387bf53b-e277-4e85-9775-93d6b5afc036" />

  
### ✔️ Dockerfile
- 이미지를 자동으로 빌드하기 위한 설정 스크립트
- 대표 명령:
  - `FROM`: 베이스 이미지 정의
  - `COPY`: 파일 복사
  - `RUN`: 빌드 단계 명령 실행
  - `CMD` / `ENTRYPOINT`: 컨테이너 실행 시 수행할 명령
  - `EXPOSE`: 문서화용 포트 표시

### ✔️ Registry
- 이미지를 저장하는 중앙 저장소  
  (Docker Hub, AWS ECR 등)

---

## 3️⃣ Dockerfile 빌드 흐름
```bash
docker build -t myapp .
docker run -d -p 8080:8080 myapp
```
👉 Dockerfile → Image → Container 순으로 동작

---

## 4️⃣ Docker 이미지 Layer 개념
- Dockerfile 명령 하나하나가 레이어가 됨
- 위 레이어가 바뀌지 않으면 아래 레이어는 캐시를 재사용 → 빠른 빌드
- `COPY` / `RUN` 등 자주 바뀌는 부분은 Dockerfile 아래쪽에 배치하는 이유

---

## 5️⃣ Docker Registry 사용 기본
```bash
docker login
docker tag myapp username/myapp:v1
docker push username/myapp:v1
docker pull username/myapp:v1
```

---

## 6️⃣ Portainer (GUI 관리 도구)
- Docker Desktop 없이도 웹 UI로 Docker 환경을 관리할 때 유용
- 컨테이너 상태, 로그, 이미지, 네트워크, 볼륨을 클릭 몇 번으로 관리
- 실무에서 원격 Docker 서버 관리할 때 자주 사용되는 패턴

설치:
```bash
docker run -d -p 9000:9000 --name portainer \
  -v /var/run/docker.sock:/var/run/docker.sock \
  portainer/portainer-ce
```

---

## 7️⃣ Volume (데이터 영속성)
- 컨테이너 삭제 시에도 데이터가 남아있는 영속 저장소
- `/var/lib/docker/volumes/...` 에 실제 저장됨
- 여러 컨테이너가 같은 볼륨 mount 가능

```bash
docker volume create myvol
docker run -v myvol:/data myapp
```

📌 여러 컨테이너가 같은 파일 동시에 수정하면?  
→ Docker가 제어하는 것이 아니라 리눅스 파일시스템 규칙을 따름  
→ "마지막으로 쓴 사람이 이김(last writer wins)"  
→ 동시 write 시 충돌/깨짐 가능 (애플리케이션에서 락 필요)  

---

## 전체 요약
| 개념              | 핵심 요약                    |
| --------------- | ------------------------ |
| Docker          | 동일한 실행 환경을 제공하는 컨테이너 플랫폼 |
| Image           | 읽기 전용 템플릿 (레이어 구조)       |
| Container       | 이미지를 실행한 프로세스            |
| Dockerfile      | 이미지 빌드 스크립트              |
| Registry        | 이미지 저장소                  |
| Volume          | 데이터 영속 저장소               |

---

## 📌 도커 기본 CLI 명령어 모음
| 명령어                | 사용법                                   | 예시                                     | 설명                     |
| ------------------ | ------------------------------------- | -------------------------------------- | ---------------------- |
| **docker run**     | `docker run [OPTIONS] IMAGE`          | `docker run -d -p 8080:80 nginx`       | 컨테이너 생성 & 실행           |
| **docker ps**      | `docker ps`                           | `docker ps -a`                         | 실행 중(또는 모든) 컨테이너 목록    |
| **docker inspect** | `docker inspect NAME/ID`              | `docker inspect mycontainer`           | 컨테이너/이미지 상세 정보(JSON)   |
| **docker logs**    | `docker logs [OPTIONS] CONTAINER`     | `docker logs -f mycontainer`           | 컨테이너 로그 보기 (`-f`: 실시간) |
| **docker exec**    | `docker exec [OPTIONS] CONTAINER CMD` | `docker exec -it mycontainer /bin/bash`     | 실행 중 컨테이너 내부에서 명령 실행   |
| **docker cp**      | `docker cp SRC DEST`                  | `docker cp mycontainer:/app/log.txt .` | 컨테이너 ↔ 호스트 간 파일 복사     |
| **docker stop**    | `docker stop CONTAINER`               | `docker stop mycontainer`              | 컨테이너 중지                |
| **docker start**   | `docker start CONTAINER`              | `docker start mycontainer`             | 중지된 컨테이너 시작            |
| **docker rm**      | `docker rm CONTAINER`                 | `docker rm -f mycontainer`             | 컨테이너 삭제 (`-f`: 강제 삭제)  |
| **-it 옵션**         | `docker run -it IMAGE`                | `docker run -it ubuntu bash`           | 인터랙티브 셸(TTY)로 실행       |
| **docker tag**     | `docker tag SOURCE TARGET`            | `docker tag app:latest jongmin/app:v1` | 이미지에 태그 붙이기            |
| **docker images**  | `docker images`                       | `docker images`                        | 로컬 이미지 목록              |
| **docker push**    | `docker push NAME:TAG`                | `docker push jongmin/app:v1`           | 레지스트리에 이미지 업로드         |
| **docker pull**    | `docker pull NAME:TAG`                | `docker pull nginx:latest`             | 레지스트리에서 이미지 다운로드       |
| **docker rmi**     | `docker rmi IMAGE`                    | `docker rmi nginx:latest`              | 로컬 이미지 삭제              |

### 많이 쓰는 명령어들
1. `docker run` – 컨테이너 만들고 실행  
2. `docker ps` – 현재 돌아가는 컨테이너 확인
3. `docker exec -it` – 컨테이너 내부로 들어가기
4. `docker logs -f` – 실시간 로그 보기
5. `docker build / tag / push` – 배포용 이미지 만들기 & 올리기

