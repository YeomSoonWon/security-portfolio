# IoT 펌웨어 보안 분석 보고서 — OWASP IoTGoat

- **분석자:** 염순원 (Sunwon Yum)
- **분석일:** 2026-11-13
- **대상:** OWASP IoTGoat v1.0 (x86), OpenWrt 기반 의도적 취약 IoT 펌웨어
- **환경:** macOS (Apple Silicon), binwalk 3.1.0 · unsquashfs 4.7.5 · QEMU 11.1.3
- **범위:** 연습용으로 공개된 취약 펌웨어에 한함. 실제 서비스/기기 대상 아님.

> 목적: 공개된 IoT 펌웨어 이미지를 추출하여 루트 파일시스템을 복원하고,
> 정적 분석으로 취약점을 식별하는 전 과정을 수행한다.

---

## 1. 분석 절차 (Methodology)

| 단계 | 도구 | 명령 |
|---|---|---|
| 1. 펌웨어 획득 | curl | IoTGoat 공식 릴리스 다운로드 |
| 2. 압축 해제 | gunzip | `.img.gz` → `.img` |
| 3. 구조 식별 | binwalk | 파티션·파일시스템 서명 스캔 |
| 4. 추출 | binwalk -e | MBR → ext4 + **Squashfs** 분리 |
| 5. 루트FS 복원 | unsquashfs | Squashfs → `rootfs/` (2,275개 파일) |
| 6. 정적 분석 | grep, strings, file, python | 자격증명·백도어·설정 점검 |

---

## 2. 주요 발견 (Findings)

### F-1. 하드코딩된 약한 비밀번호 해시 — **High**
`/etc/shadow`에 root와 iotgoatuser의 비밀번호가 **MD5-crypt(`$1$`)** 로 저장됨.
MD5-crypt는 GPU로 초당 수억 회 대입이 가능해 사전 공격에 매우 취약.

```
root:$1$Jl7H1VOG$Wgw2F/C.nLNTC.4pwDa4H1:...
iotgoatuser:$1$79bz0K8z$Ii6Q/if83F1QodGmkb4Ah.:...
```
- **영향:** 펌웨어만 입수하면 오프라인에서 비밀번호 복원 가능 → 기기 완전 장악.
- **권고:** SHA-512-crypt(`$6$`) 이상 또는 yescrypt 사용, 디바이스별 고유 비밀번호.

### F-2. 내장 백도어 바이너리 `shellback` — **Critical**
`/usr/bin/shellback` (ELF 32-bit i386). 문자열·호출 구조 분석 결과 **바인드 셸 백도어**.
- `socket → bind → listen → fork → dup2 → execve(/bin/busybox sh)` 구조 확인.
- 포트 상수 **65535** 가 바이너리 내 htons(빅엔디안) 형태로 존재.
- 접속 성공 시 배너 출력: `[***]Successfully Connected to IoTGoat's Backdoor[***]`
- **영향:** 65535 포트 접속만으로 인증 없이 루트 셸 획득.
- **권고:** 미승인 리스닝 서비스 제거, 펌웨어 서명·무결성 검증 도입.

### F-3. 원격 접속(dropbear) 위험 설정 — **Medium**
`/etc/config/dropbear`에서 SSH가 **비밀번호 인증 + root 직접 로그인** 모두 허용.
```
option PasswordAuth 'on'
option RootPasswordAuth 'on'
```
- **영향:** F-1의 약한 해시와 결합 시 SSH 무차별 대입으로 root 접속 가능.
- **권고:** `RootPasswordAuth 'off'`, 키 기반 인증 전환.

### F-4. 설정 파일 내 평문 비밀번호 참조 — **Low~Medium**
`/etc/config/rpcd`에 `option password '$p$root'` 형태로 root 해시를 참조하는 구조 발견.
서비스 설정에서 자격증명을 직접 끌어오는 패턴으로, 설정 노출 시 추가 공격면이 됨.

---

## 3. 요약

| ID | 발견 | 심각도 | 근거 |
|---|---|---|---|
| F-1 | 약한 MD5 비밀번호 해시 | High | `/etc/shadow` |
| F-2 | 내장 바인드 셸 백도어 (65535) | Critical | `/usr/bin/shellback` |
| F-3 | SSH root 비밀번호 로그인 허용 | Medium | `/etc/config/dropbear` |
| F-4 | 설정 내 비밀번호 참조 | Low~Med | `/etc/config/rpcd` |

**공고 대응:** 본 분석은 "(A)IoT/임베디드 취약점 분석", "바이너리 기반 취약점 분석",
"소스/설정 기반 취약점 분석" 직무 요건에 직접 대응함.

## 4. 다음 단계
- [ ] `shellback` Ghidra 디컴파일로 포트·인증 로직 정밀 확인
- [ ] john/hashcat로 F-1 해시 실제 크랙 재현 (연습 범위 내)
- [ ] QEMU로 펌웨어 부팅 후 65535 포트 백도어 동적 재현
