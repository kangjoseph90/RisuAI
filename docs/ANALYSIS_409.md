# Node.js 환경(RisuAI Server)에서의 409 Conflict 분석

본 문서는 RisuAI Node.js 서버 환경에서 HTTP 409 (Conflict) 에러가 발생하는 원인과 메커니즘을 분석한 내용입니다.

## 1. 개요

RisuAI 서버에서 `409 Conflict`는 주로 **데이터 동기화(Patch Sync)** 과정에서 클라이언트와 서버 간의 데이터 상태가 불일치할 때 발생합니다. 이는 버그가 아니라 데이터 무결성을 보장하기 위한 **낙관적 락(Optimistic Locking)** 메커니즘의 의도된 동작입니다.

## 2. 발생 메커니즘

서버 코드(`server/node/server.cjs`)의 `/api/patch` 엔드포인트에서 충돌 감지 로직이 구현되어 있습니다.

### 충돌 감지 로직
1.  **요청 수신**: 클라이언트가 서버로 패치 요청을 보낼 때, 변경 사항(`patch`)과 함께 클라이언트가 알고 있는 현재 데이터의 해시(`expectedHash`)를 전송합니다.
2.  **서버 상태 확인**: 서버는 요청받은 파일(`filePath`)에 해당하는 데이터를 메모리 캐시(`dbCache`)에서 조회하거나 디스크에서 로드합니다.
3.  **해시 계산**: 서버는 현재 보유한 데이터의 해시값(`serverHash`)을 `server/node/utils.cjs`의 `calculateHash` 함수를 통해 계산합니다.
4.  **비교 및 검증**:
    ```javascript
    // server/node/server.cjs (Line 352 approx)
    if (expectedHash !== serverHash) {
        console.log(`[Patch] Hash mismatch...`);
        res.status(409).send({
            error: 'Hash mismatch - data out of sync',
        });
        return;
    }
    ```
    만약 클라이언트가 보낸 해시와 서버가 계산한 해시가 다르면, 클라이언트가 오래된 데이터를 기반으로 수정을 시도하는 것으로 간주하여 `409` 에러를 반환합니다.

## 3. 주요 발생 시나리오

### 3.1. 동시 편집 (Concurrent Editing)
두 클라이언트(또는 두 탭)가 같은 파일을 열어두고 있는 상황:
1.  **User A**가 데이터를 수정하고 저장을 시도 -> 서버 상태 업데이트 완료 (해시 변경됨).
2.  **User B**가 수정 전 상태에서 데이터를 수정하고 저장을 시도.
3.  User B의 클라이언트는 구버전 해시를 `expectedHash`로 전송.
4.  서버의 현재 해시와 불일치 -> **409 발생**.

### 3.2. 캐시와 디스크 간의 경쟁 상태 (Race Condition)
RisuAI 서버는 메모리 기반의 패치 시스템(`dbCache`)과 디스크 저장(`fs.writeFile`)을 병행합니다.
- `/api/write` (전체 덮어쓰기)가 호출되면 `dbCache`가 초기화됩니다.
- 만약 `/api/write` 직후 다른 클라이언트가 이전에 캐시된 상태를 기반으로 `/api/patch`를 요청하면, 서버는 디스크에서 최신 파일(또는 정규화된 파일)을 다시 로드합니다.
- 이때 미세한 데이터 차이(정규화 과정 등)나 버전 차이로 인해 해시가 달라져 **409 발생** 가능성이 있습니다.

### 3.3. Node.js 버전 호환성 문제 (간접적 원인)
`server/node/server.cjs`에는 Node.js v22.7.0 및 v23+ 버전에서 `msgpackr` 버그로 인한 충돌을 방지하기 위해 패치 동기화를 비활성화하는 코드가 있습니다.
```javascript
if (major >= 23 || (major === 22 && minor === 7 && patch === 0)) {
    enablePatchSync = false;
}
```
이 경우 409 대신 `404 Not Found` (Patch sync is not enabled)가 발생하겠지만, 환경에 따라 비정상적인 동작이 동기화 오류로 이어질 수 있습니다.

## 4. 해시 계산의 정합성

`server/node/utils.cjs`의 `calculateHash` 함수는 객체의 키 순서와 상관없이 일관된 해시를 생성하도록 설계되어 있습니다(덧셈 연산 사용). 그러나 `normalizeJSON` 함수를 통해 데이터를 정규화하는 과정에서 `undefined`, `null`, `Date` 객체 등의 처리가 달라질 경우, 논리적으로 같은 데이터라도 해시 불일치가 발생할 수 있습니다.

## 5. 결론

RisuAI 환경에서 409 에러는 **"데이터가 동기화되지 않았음"**을 알리는 신호입니다. 이를 해결하기 위해 클라이언트는 최신 데이터를 다시 `fetch`한 후 작업을 재시도해야 합니다.
