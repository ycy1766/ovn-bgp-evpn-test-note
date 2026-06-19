# OVN 기여 가이드: 버그 수정 및 기능 추가 PR 제출 방법

> OVN은 **GitHub PR이 아니라 ovs-dev 메일링리스트(`dev@openvswitch.org`)로 패치를 보내** 리뷰·머지한다.
> GitHub PR에는 CI만 돌고 실제 머지는 리스트에서 이뤄진다.
> 아래는 메일 환경 설정부터 발송·리뷰 대응까지 전 과정.

---

## 0. 핵심 요약 (TL;DR)

| 항목 | 규칙 |
|---|---|
| 제출 경로 | ovs-dev 메일링리스트 (`git send-email`) |
| **제목 태그** | **반드시 `[PATCH ovn vN n/m]`** — `ovn` 빠지면 robot이 OVS 레포에 적용 시도 → 실패 |
| 서명 | `Signed-off-by:` (DCO) 필수, **author 이름과 정확히 일치** |
| 제목 길이 | `[PATCH ovn vN n/m]` 포함 **70자 이내** |
| 검증 | `make check` + `checkpatch.py` 통과 |
| 베이스 | 최신 `upstream/main` 위로 rebase, `--base` 로 base-commit 기록 |

전체 흐름:

```
[1회] 메일환경 설정  →  저장소 준비  →  코드 작업 & 커밋(DCO)
  →  upstream rebase  →  빌드 & make check  →  checkpatch
  →  format-patch(+cover letter, --subject-prefix="PATCH ovn")
  →  git send-email (RFC 있으면 --in-reply-to)  →  robot/리뷰 대응(v2,v3..)
```

---

## 1. 메일 환경 설정 (`git send-email`, 1회)

### 1.1 git-email 설치
```bash
# Ubuntu/Debian
sudo apt install -y git-email
# macOS (Homebrew git에 포함)
brew install git
```

### 1.2 Gmail 앱 비밀번호 발급 (웹에서)
Gmail 일반 비밀번호로는 SMTP 인증이 안 된다. **앱 비밀번호 16자리**를 발급한다.

1. **2단계 인증 ON** (전제조건): <https://myaccount.google.com/security>
2. **앱 비밀번호 생성**: <https://myaccount.google.com/apppasswords>
   - 앱 이름 입력(예: `git-send-email`) → 만들기
   - 표시되는 **16자리**(예: `abcd efgh ijkl mnop`)를 공백 제거해 사용 (`abcdefghijklmnop`)
   - 창 닫으면 다시 안 보이니 즉시 복사
   > "앱 비밀번호" 메뉴가 없으면 → 2단계 인증이 꺼져 있거나 조직(워크스페이스) 계정이라 막힌 경우.

### 1.3 `~/.gitconfig` 설정
```ini
[user]
    name = Hong Gildong
    email = you@gmail.com
[sendemail]
    smtpserver = smtp.gmail.com
    smtpencryption = tls
    smtpserverport = 587
    smtpuser = you@gmail.com
    # smtppass = <앱비밀번호>   ; 비워두면 보낼 때 프롬프트로 물어봄(권장)
    confirm = always
```
> SMTP 인자를 명령에 직접 줘도 된다(아래 8장 참고).

---

## 2. 저장소 준비 (1회)

```bash
git clone https://github.com/ovn-org/ovn.git
cd ovn
git remote rename origin upstream          # ovn-org 를 upstream 으로
git remote add origin https://github.com/<나>/ovn.git   # 내 fork(선택)
git submodule update --init                # OVS 소스(빌드에 필요)

# ★ OVN 패치 제목 태그를 자동으로 'ovn' 으로
git config format.subjectPrefix "PATCH ovn"

# 빌드 의존성 (Ubuntu/Debian)
sudo apt install -y build-essential autoconf automake libtool \
    libssl-dev libcap-ng-dev python3 python3-sphinx
```

---

## 3. 코드 작업 & 커밋

```bash
git fetch upstream main
git checkout -b <feature> upstream/main
# ... 코드 수정 ...
git add <files>
git commit -s        # -s = Signed-off-by 자동 추가 (DCO 필수)
```

### 커밋 메시지 규칙
- **제목**: `area: 요약.` 형식 (예: `northd: Advertise distributed NAT IPs over EVPN.`)
  - **`[PATCH ovn vN n/m]` 접두어까지 합쳐 70자 이내** → 요약은 대략 **50자 이내**로
- **본문**: *무엇이 아니라 왜*. 한 줄 ≤ 75자, 명령형 현재시제
- **트레일러**:
  ```
  Reported-by: ...
  Suggested-by: ...
  Signed-off-by: Hong Gildong <you@gmail.com>
  ```
- ⚠️ **Author 와 Signed-off-by 가 대소문자까지 정확히 일치**해야 함
  - 안 맞으면 checkpatch `ERROR: Author ... needs to sign off`
  - git 이름 맞추기: `git config user.name "Hong Gildong"`
  - 이미 커밋했으면 일괄 교정:
    ```bash
    git rebase upstream/main \
      --exec 'git commit --amend --no-edit --author="Hong Gildong <you@gmail.com>"'
    ```
- 논리 단위로 커밋 분리(예: `northd:` / `controller:`). 의존 커밋은 본문에서 상대 커밋 제목 인용.

---

## 4. upstream/main 위로 rebase

회사 fork 등 다른 베이스로 작업했다면 업스트림 위로 옮긴다.
```bash
git fetch upstream main
git rebase upstream/main
# 충돌 시: 수정 → git add → git rebase --continue
```
확인:
```bash
git log --oneline upstream/main..HEAD    # 내 커밋만
git diff --stat upstream/main            # PR diff 미리보기
```

---

## 5. 빌드 & 테스트 (필수)

```bash
./boot.sh && ./configure && make -j4
make check TESTSUITEFLAGS="-k '<관련 키워드>'"   # 관련 테스트만
make check                                     # 전체(오래 걸림)
```
> macOS에서는 시스템 netdev(br-phys) 필요 테스트나 `table=??` 정규화 테스트가
> 환경 문제로 실패할 수 있다 — 내 변경과 무관한지 구분.

---

## 6. checkpatch (스타일 검사)

```bash
python3 utilities/checkpatch.py /tmp/patches/00*.patch
# 각 패치 "no obvious problems found" 나와야 함
```
자주 걸림: 제목 70자 초과, **author≠signoff**, trailing whitespace, 80컬럼 초과.

---

## 7. 패치 생성 (format-patch + cover letter)

```bash
mkdir -p /tmp/patches
git format-patch -v1 --subject-prefix="PATCH ovn" --cover-letter \
    --base=upstream/main -o /tmp/patches upstream/main
```
- **`--subject-prefix="PATCH ovn"`** → 제목이 `[PATCH ovn vN n/m]` (2장 config 했으면 생략 가능)
- **`--base=upstream/main`** → patch에 `base-commit:` 기록(적용 기준 명시)
- 패치 2개↑면 cover letter(0000) 권장. `*** SUBJECT HERE ***`/`*** BLURB HERE ***` 채우기
  (문제·설계·테스트·RFC 링크, 다음 버전이면 "Changes since vN")

---

## 8. 발송 (`git send-email`)

```bash
# (1) dry-run — 실제 발송 안 함, 헤더/수신자만 확인
git send-email --dry-run --no-annotate --confirm=never \
  --from="Hong Gildong <you@gmail.com>" \
  --to=dev@openvswitch.org \
  /tmp/patches/*.patch

# (2) 실제 발송 (SMTP 인자 직접 지정 + 앱비밀번호 프롬프트)
git send-email --no-annotate --confirm=never \
  --smtp-server=smtp.gmail.com --smtp-encryption=tls --smtp-server-port=587 \
  --smtp-user=you@gmail.com \
  --from="Hong Gildong <you@gmail.com>" \
  --to=dev@openvswitch.org \
  /tmp/patches/*.patch
```
- 실행하면 `Password for 'smtp://...':` → **앱 비밀번호 16자리** 붙여넣기
- 각 통이 SMTP **`Result: 250`** 이면 성공
- `Suggested-by:`/`Reported-by:` 의 사람은 **자동 Cc** 됨

### 기존 RFC 스레드에 이어붙이기
RFC를 먼저 올려 메인테이너 답을 받았다면, 그 메일의 **Message-ID**로 스레드 연결:
```bash
git send-email ... --in-reply-to='<원본메일-Message-ID>' /tmp/patches/*.patch
```
- Message-ID 찾기: Gmail 보낸편지함 → 해당 메일 → ⋮ → **"원본 보기"** → `Message-ID:` 헤더
  (또는 ovs-dev 아카이브/patchwork)

---

## 9. 0-day Robot & 리뷰 대응

- 발송 후 **0-day Robot**이 자동으로 patch를 적용·빌드·checkpatch 실행, 결과를 메일로 회신
- patchwork 에서 상태 추적: <https://patchwork.ozlabs.org/project/ovn/list/>
- 리뷰 코멘트 반영 후 **버전 올려 재발송**:
  ```bash
  git format-patch -v2 --subject-prefix="PATCH ovn" --cover-letter \
      --base=upstream/main -o /tmp/patches upstream/main
  git send-email ... --in-reply-to='<v1 cover Message-ID>' /tmp/patches/*.patch
  ```
- cover letter나 각 패치 `---` 아래에 **변경 이력** 기재:
  ```
  Changes since v1:
   - en_northd 핸들러를 incremental 로 변경, 테스트 보강.
  ```

---

## 10. 자주 겪는 함정 (트러블슈팅)

| 증상 | 원인 | 해결 |
|---|---|---|
| robot `git-am: could not build fake ancestor` | **제목에 `ovn` 태그 없음** → OVS 레포에 적용 시도 | `--subject-prefix="PATCH ovn"` 로 재발송 |
| checkpatch `Author ... needs to sign off` | author 이름 ≠ Signed-off-by | `git config user.name` 맞추고 `--author` 로 amend |
| checkpatch `subject over 70 characters` | `[PATCH ovn vN n/m]` 포함 70자 초과 | 요약을 ~50자 이내로 단축 |
| SMTP 인증 실패 | Gmail 일반 비번 사용 | **앱 비밀번호** 발급해 사용 |
| patch가 robot에만 깨짐 | (대개) 위 태그 문제 | 로컬에서 `git am` 되는지 확인(되면 전송/태그 문제) |

> robot이 못 막는데 로컬 `git worktree add /tmp/t upstream/main && cd /tmp/t && git am <patch>` 로
> 깨끗이 적용되면, 패치 자체는 정상이고 **태그/전송/robot base** 쪽 문제다.

---

## 부록 A. 이번 EVPN Type-2 FIP 패치 실제 값

| 항목 | 값 |
|---|---|
| 업스트림 브랜치 | `evpn-fip-type2-upstream` (ovn-org/main 위 2커밋), `github.com/ycy1766/ovn` |
| 회사 fork PR | `github.com/kt-cloud-stack/ovn/pull/1` |
| 커밋1 | `northd: Advertise distributed NAT IPs over EVPN.` |
| 커밋2 | `controller: Add Advertised_MAC_Binding to FDB.` |
| RFC 스레드 | ovs-dev "EVPN: advertised_mac doesn't include NAT dnat_and_snat entries" |
| 검증 | Kube-OVN/OVN 26.03.90 랩, FIP advertised_mac 등록 + GW ping 0% loss |
| **교훈** | v1~v3는 제목에 `ovn` 누락 → robot 실패. **v4에서 `[PATCH ovn]` 로 해결** |

## 부록 B. 자주 쓰는 한 줄

```bash
# author 일괄 교정
git rebase upstream/main --exec 'git commit --amend --no-edit --author="Hong Gildong <you@gmail.com>"'

# 패치 재생성 + checkpatch + 로컬 am 검증을 한 번에
rm -rf /tmp/patches && mkdir /tmp/patches && \
git format-patch -v2 --subject-prefix="PATCH ovn" --cover-letter --base=upstream/main -o /tmp/patches upstream/main && \
python3 utilities/checkpatch.py /tmp/patches/v2-000[12]*.patch && \
git worktree add -q /tmp/amtest upstream/main && (cd /tmp/amtest && git am /tmp/patches/v2-000[12]*.patch) ; \
git worktree remove --force /tmp/amtest
```
