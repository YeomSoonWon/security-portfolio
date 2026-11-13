# 재현 방법 — IoTGoat 분석

> 펌웨어 원본은 저장소에 없습니다. 아래 순서로 직접 받아 재현하세요.

```bash
# 0. 도구 (macOS)
brew install binwalk qemu squashfs

# 1. 펌웨어 다운로드 (OWASP IoTGoat 공식 릴리스)
mkdir -p firmware && cd firmware
curl -L -o IoTGoat-x86.img.gz \
  https://github.com/OWASP/IoTGoat/releases/download/v1.0/IoTGoat-x86.img.gz
gunzip -k IoTGoat-x86.img.gz

# 2. 구조 식별 + 추출 (MBR → ext4 + Squashfs)
binwalk IoTGoat-x86.img
binwalk -e IoTGoat-x86.img

# 3. 루트 파일시스템 복원 (두 번째 파티션이 Squashfs)
unsquashfs -d rootfs \
  extractions/IoTGoat-x86.img.extracted/0/Linux_partition.1

# 4. 정적 분석 예시
cat rootfs/etc/shadow                       # F-1 약한 MD5 해시
file rootfs/usr/bin/shellback               # F-2 백도어 바이너리
strings rootfs/usr/bin/shellback | less     # 백도어 문자열/포트
cat rootfs/etc/config/dropbear              # F-3 SSH root 로그인
```

분석 결과는 [REPORT.md](REPORT.md) 참고.
