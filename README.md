# vdsm

# TrueNAS + Portainer + Virtual DSM(vdsm) 구축 매뉴얼

> TrueNAS Scale 환경에서 `vdsm/virtual-dsm`을 이용하여  
> Synology DSM을 Docker 기반으로 구축하고,
>
> - SMB 공유 연동
> - 자동 마운트
> - macvlan 독립 IP
> - 외부 FTP(FileZilla) 연결
>
> 까지 구성하는 전체 매뉴얼.

---

# 최종 구성도

```text
인터넷
 ↓
공유기
 ↓
TrueNAS (192.168.0.5)
 └ Portainer
    └ Virtual DSM (192.168.0.241)
```

---

# 목표

Virtual DSM이:

- 독립 NAS처럼 동작
- FTP / SMB / DDNS 사용 가능
- 외부 FileZilla 접속 가능

하도록 구성.

---

# 1. TrueNAS SMB 공유 생성

TrueNAS에서 SMB 공유 생성.

예시 경로:

```text
/mnt/Theh_1
```

SMB Share 이름:

```text
SMB_Share_server1
```

---

# 2. Portainer 설치

TrueNAS Apps 또는 Docker로 Portainer 설치.

접속 예시:

```text
http://192.168.0.5:31015
```

---

# 3. macvlan 네트워크 생성

TrueNAS Shell 접속.

---

## 3-1. NIC 이름 확인

```bash
ip addr
```

예시:

```text
eno3
```

---

## 3-2. macvlan 생성

```bash
docker network create -d macvlan \
  --subnet=192.168.0.0/24 \
  --gateway=192.168.0.1 \
  --ip-range=192.168.0.240/28 \
  -o parent=eno3 \
  dsm_macvlan
```

---

# 4. Virtual DSM Stack 생성

Portainer → `Stacks` → `Add Stack`

---

## docker-compose.yml

```yaml
services:
  vdsm:
    image: vdsm/virtual-dsm:latest
    container_name: virtual-dsm

    environment:
      DISK_SIZE: "500G"
      RAM_SIZE: "8G"
      CPU_CORES: "4"

    devices:
      - /dev/kvm
      - /dev/net/tun

    cap_add:
      - NET_ADMIN

    networks:
      dsm_macvlan:
        ipv4_address: 192.168.0.241

    volumes:
      - /mnt/Theh_1/docker/vdsm:/storage

    restart: unless-stopped

    stop_grace_period: 2m

networks:
  dsm_macvlan:
    external: true
```

---

# 5. DSM 설치

브라우저 접속:

```text
http://192.168.0.241:5000
```

DSM 초기 설치 진행.

---

# 6. DSM SSH 활성화

DSM:

```text
제어판
→ 터미널 및 SNMP
→ SSH 서비스 활성화
```

---

# 7. DSM에 TrueNAS SMB 공유 마운트

DSM SSH 접속:

```bash
ssh hvfxprod@192.168.0.241
```

root 권한 획득:

```bash
sudo su
```

---

## 7-1. 마운트 경로 생성

```bash
mkdir -p /volume1/SMB_Share
```

---

## 7-2. SMB Credential 파일 생성

```bash
vi /root/.smbcred
```

내용:

```text
username=트루나스유저명
password=트루나스비밀번호
```

권한 설정:

```bash
chmod 600 /root/.smbcred
```

---

## 7-3. SMB 수동 마운트 테스트

```bash
mount -t cifs //192.168.0.5/SMB_Share_server1 /volume1/SMB_Share \
-o credentials=/root/.smbcred,vers=3.0
```

---

## 7-4. 확인

```bash
ls -la /volume1/SMB_Share
```

---

# 8. DSM 재부팅 시 자동 마운트

---

## 8-1. 자동 마운트 스크립트 생성

```bash
vi /usr/local/etc/rc.d/truenas_mount.sh
```

내용:

```bash
#!/bin/sh

mkdir -p /volume1/SMB_Share

mount -t cifs //192.168.0.5/SMB_Share_server1 /volume1/SMB_Share \
-o credentials=/root/.smbcred,vers=3.0
```

---

## 8-2. 실행 권한 부여

```bash
chmod +x /usr/local/etc/rc.d/truenas_mount.sh
```

---

## 8-3. 테스트 실행

```bash
/usr/local/etc/rc.d/truenas_mount.sh
```

---

# 9. DSM homes 경로 연결

DSM FTP / File Station 호환을 위한 homes 경로 생성.

---

## 9-1. homes 경로 생성

```bash
mkdir -p /volume1/homes/hvfxprod
```

---

## 9-2. SMB 링크 생성

```bash
ln -s /volume1/SMB_Share /volume1/homes/hvfxprod/SMB_Share
```

---

## 9-3. 권한 설정

```bash
chmod 700 /volume1/homes/hvfxprod
```

---

# 10. FTP 활성화

DSM:

```text
제어판
→ 파일 서비스
→ FTP
```

설정:

```text
FTP 활성화: ON
FTP 포트: 1014

PASV 모드 외부 IP 보고: ON

Passive Range:
55550 ~ 55560
```

---

# 11. 공유기 포트포워딩

---

## 11-1. FTP Control Port

```text
외부 1014
→ 192.168.0.241:1014
```

---

## 11-2. Passive Port

```text
55550~55560
→ 192.168.0.241
```

TCP 기준.

---

# 12. DDNS 연결

예시:

```text
fileshare.hvfxprod.com
```

DNS 구조:

```text
fileshare.hvfxprod.com
→ hvfxnas.iptime.org
→ 공인IP
```

---

# 13. FileZilla 테스트

호스트:

```text
fileshare.hvfxprod.com
```

포트:

```text
1014
```

암호화:

```text
일반 FTP 사용
```

---

# 최종 결과

구축 완료 시:

- ✅ DSM 독립 IP 사용
- ✅ 외부 FTP 접속 가능
- ✅ SMB 자동 마운트
- ✅ 재부팅 후 자동 연결
- ✅ File Station 연동
- ✅ macvlan 기반 독립 NAS 구성 완료

---

# 참고 사항

## macvlan 특징

macvlan 사용 시:

```text
TrueNAS Host ↔ DSM
```

직접 통신이 제한될 수 있음.

하지만:

- 외부 PC
- 같은 LAN 장치

에서는 정상 접근 가능.

---

# 추천 추가 기능

향후 추가 가능:

- FTPS
- WebDAV
- Synology Drive
- 외부 SMB
- Cloudflare DDNS
- Nginx Proxy Manager HTTPS Reverse Proxy

등.

---

# Troubleshooting

---

## FTP 로그인은 되는데 디렉토리 조회 실패

증상:

```text
MLSD
20초간 활동이 없어 연결 시간이 만료됨
```

원인:

```text
Passive FTP 포트 미포워딩
```

해결:

```text
55550~55560 TCP
→ DSM IP 포트포워딩
```

---

## FTP 접속 시 530 Login incorrect

원인 가능성:

- TrueNAS FTP와 포트 충돌
- DSM homes 경로 문제
- Docker bridge NAT 문제

해결:

- TrueNAS FTP 비활성화
- macvlan 적용
- homes 경로 생성

---

## DSM이 외부 FTP에 응답하지 않음

확인 사항:

- DSM FTP 활성화 여부
- 공유기 포트포워딩
- DSM macvlan IP 확인
- PASV 외부 IP 설정 확인

---