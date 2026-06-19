# OVN 업스트림 패치 컨트리뷰션 가이드 (ovs-dev)

OVN은 **GitHub PR이 아니라 ovs-dev 메일링리스트(`dev@openvswitch.org`)로 패치를 보내** 리뷰·머지한다.
GitHub PR에는 CI만 돌고 머지는 리스트에서 이뤄진다. 아래는 우리가 EVPN Type-2 FIP 패치를
실제로 준비한 전 과정(복붙용).

---

## 0. 한눈에 보는 흐름

```
브랜치 작업 → 커밋(DCO 서명) → upstream/main 위로 rebase
→ 빌드 & make check → checkpatch → format-patch(+cover letter)
→ git send-email (RFC 스레드에 --in-reply-to)
```

---

## 1. 사전 준비 (1회)

```bash
# upstream 리모트 추가
git remote add upstream https://github.com/ovn-org/ovn.git
git fetch upstream main

# 빌드 의존성 (Ubuntu/Debian)
sudo apt install -y build-essential autoconf automake libtool \
    libssl-dev libcap-ng-dev python3 python3-sphinx git-email
# OVS 소스 필요 (서브모듈)
git submodule update --init
```

---

## 2. 코드 작업 & 커밋 규칙

```bash
git checkout -b <feature-branch> upstream/main
# ... 코드 수정 ...
git add <files>
git commit -s          # -s = Signed-off-by 자동 추가 (DCO 필수)
```

### 커밋 메시지 규칙 (checkpatch가 강제)
- **Subject ≤ 70자**, `area: 요약.` 형식 (예: `northd: Advertise distributed NAT IPs/MACs over EVPN.`)
- 본문은 **무엇이 아니라 왜**를 설명, 한 줄 ≤ 75자, 명령형 현재시제
- 끝에 트레일러:
  ```
  Reported-by: ...
  Suggested-by: ...
  Signed-off-by: Hong Gildong <you@example.com>
  ```
- ⚠️ **Author 와 Signed-off-by 가 정확히 일치**해야 함 (대소문자 포함).
  안 맞으면 checkpatch `ERROR: Author ... needs to sign off`.
  - git 사용자명 맞추기: `git config user.name "Hong Gildong"`
  - 이미 커밋했다면 author 교정:
    ```bash
    git rebase upstream/main \
      --exec 'git commit --amend --no-edit --author="Hong Gildong <you@example.com>"'
    ```

> 커밋 여러 개를 논리 단위로 분리한다 (예: northd 변경 / controller 변경).
> 컨트롤러 커밋 본문에서 northd 커밋 제목을 인용하면 연관성이 드러난다.

---

## 3. upstream/main 위로 rebase

회사 fork(kt-cloud-stack 등) 베이스로 작업했다면 업스트림 위로 옮긴다.

```bash
git fetch upstream main
git checkout -b <feature>-upstream <feature-branch>
git rebase upstream/main
# 충돌 나면: 파일 수정 → git add → git rebase --continue
```

확인:
```bash
git log --oneline upstream/main..HEAD          # 내 커밋만 떠야 함
git diff --stat upstream/main                  # PR diff 미리보기
```

---

## 4. 빌드 & 테스트 (보내기 전 필수)

```bash
./boot.sh && ./configure && make -j4
# 관련 테스트만
make check TESTSUITEFLAGS="-k 'distributed NAT'"
make check TESTSUITEFLAGS="-k 'graph'"
# 전체 (시간 걸림)
make check
```
> macOS에서는 시스템 netdev(br-phys) 필요 테스트나 `table=??` 정규화 테스트가
> 환경 문제로 깨질 수 있다 — 내 변경과 무관한지 구분할 것.

---

## 5. checkpatch (스타일 검사)

```bash
python3 utilities/checkpatch.py /tmp/patches/00*.patch
# "no obvious problems found" 나와야 함
```
자주 걸리는 것: subject 70자 초과, author≠signoff, trailing whitespace,
80컬럼 초과, tab/space 혼용.

---

## 6. 패치 생성 (+ cover letter)

> ⚠️ **OVN 패치는 반드시 제목 태그에 `ovn` 을 넣어야 한다** → `[PATCH ovn vN n/m]`.
> ovs-dev 리스트는 OVS/OVN을 같이 받고, **0-day robot이 제목의 `ovn` 태그로 어느 레포에
> 적용할지 결정**한다. `ovn` 이 없으면 robot이 OVS 트리에 적용하려다
> `could not build fake ancestor` 로 실패한다(실제로 v1~v3가 이 이유로 다 실패함).
> 이 repo 기본값으로 박아두면 편하다:
> ```bash
> git config format.subjectPrefix "PATCH ovn"
> ```

```bash
mkdir -p /tmp/patches
# -v2/-v3.. 는 재발송 버전. --subject-prefix 로 ovn 태그 강제(또는 위 config).
git format-patch -v2 --subject-prefix="PATCH ovn" --cover-letter \
    --base=upstream/main -o /tmp/patches upstream/main
```
> `--base=` 는 patch에 base-commit 을 기록해 적용 기준을 명시한다(권장).
생성물:
```
0000-cover-letter.patch   ← *** SUBJECT HERE *** / *** BLURB HERE *** 채워야 함
0001-....patch
0002-....patch
```
cover letter(0000)에 `[PATCH 0/N] 제목`과 본문(문제/설계/테스트/RFC 링크)을 작성.
2개 이상 패치 시리즈면 cover letter 권장, 단일 패치면 생략 가능.

---

## 7. git send-email 설정 (Gmail 기준)

### (1) Gmail 앱 비밀번호 발급
- Google 계정 → 보안 → 2단계 인증 ON → **앱 비밀번호** 생성(16자리).
  일반 비밀번호로는 SMTP 인증 안 됨.

### (2) ~/.gitconfig 설정
```ini
[sendemail]
    smtpserver = smtp.gmail.com
    smtpencryption = tls
    smtpserverport = 587
    smtpuser = you@gmail.com
    # smtppass = <앱비밀번호>   # 안 넣으면 보낼 때 물어봄(권장)
    confirm = always
    annotate = yes
[user]
    name = Hong Gildong
    email = you@gmail.com
```

### (3) 받는 사람 확인
```bash
# 메인테이너/리뷰어 추천
scripts/get_maintainer.pl /tmp/patches/0001-*.patch   # (있으면)
```
기본: `--to=dev@openvswitch.org`, 필요 시 관련 메인테이너를 `--cc`.

### (4) 보내기 (테스트 먼저)
```bash
# 진짜 안 보내고 출력만 확인
git send-email --dry-run --to=dev@openvswitch.org /tmp/patches/*.patch

# 실제 발송
git send-email --to=dev@openvswitch.org /tmp/patches/*.patch
```

### (5) 기존 RFC 스레드에 이어붙이기 (중요)
이미 RFC를 올려 메인테이너 답을 받았다면, 그 메일의 **Message-ID**를 찾아
스레드로 연결한다 (lore/patchwork에서 원문 메일 헤더의 `Message-ID:` 확인).
```bash
git send-email --to=dev@openvswitch.org \
  --in-reply-to='<RFC메일의-Message-ID@...>' \
  /tmp/patches/*.patch
```

---

## 8. 리뷰 대응 & 다음 버전(v2)

- 리뷰 코멘트 반영 후 커밋 수정/추가.
- 버전 올려 재발송:
  ```bash
  git format-patch --cover-letter -v2 -o /tmp/patches upstream/main
  ```
- cover letter나 각 패치의 `---` 아래에 **변경 이력** 적기:
  ```
  ---
  v2: en_northd 핸들러를 incremental 로 변경, 테스트 보강.
  ```
- 같은 스레드 유지 위해 v1 메일에 `--in-reply-to`.

---

## 부록 A. 이번 EVPN Type-2 FIP 패치 실제 값

| 항목 | 값 |
|---|---|
| 업스트림 브랜치 | `evpn-fip-type2-upstream` (ovn-org/main 위 2커밋) |
| 개인 포크 | `github.com/ycy1766/ovn` (push 됨) |
| 회사 fork PR | `github.com/kt-cloud-stack/ovn/pull/1` |
| 커밋1 | `northd: Advertise distributed NAT IPs/MACs over EVPN.` |
| 커밋2 | `controller: Add Advertised_MAC_Binding to EVPN FDB.` |
| 검증 | Kube-OVN/OVN 26.03.90 랩, FIP advertised_mac 등록 + GW ping 0% loss |

## 부록 B. 자주 쓰는 한 줄

```bash
# author 일괄 교정(시리즈 전체)
git rebase upstream/main --exec 'git commit --amend --no-edit --author="Hong Gildong <you@gmail.com>"'

# 패치 재생성 + checkpatch 한 번에
rm -rf /tmp/patches && mkdir /tmp/patches && \
git format-patch --cover-letter -o /tmp/patches upstream/main && \
python3 utilities/checkpatch.py /tmp/patches/00[12]*.patch
```
