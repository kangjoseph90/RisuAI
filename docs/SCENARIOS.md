# RisuAI 409 Conflict 발생 시나리오 분석

본 문서는 RisuAI의 데이터 동기화 과정에서 `409 Conflict` (Hash Mismatch)를 유발할 수 있는 계산 편차 시나리오들을 분석한 결과입니다. 실험을 통해 확인된 내용과 이론적 가능성을 포함합니다.

## 1. 개요

RisuAI는 클라이언트(브라우저)와 서버(Node.js) 양측에서 `calculateHash(normalizeJSON(data))` 함수를 통해 데이터의 해시를 계산하고 이를 비교하여 동기화 상태를 확인합니다. 이 과정에서 두 환경의 미세한 차이로 인해 **논리적으로 같은 데이터임에도 불구하고 다른 해시가 생성되는 경우**, 동기화 오류가 발생합니다.

## 2. 실험 결과 요약

| 시나리오 | 설명 | 결과 | 위험도 |
| :--- | :--- | :---: | :---: |
| **Binary Data** | Node.js `Buffer` vs Browser `Uint8Array` | 일치 | 낮음 |
| **Float Precision** | JSON 직렬화/역직렬화에 따른 부동소수점 변화 | 일치 | 낮음 |
| **Sparse Arrays** | 배열 내 `undefined` 포함 시 처리 | 일치 | 낮음 |
| **String Object** | `new String("text")` vs `"text"` | **불일치** | **중간** |
| **Prototype Props** | 객체 프로토타입 속성 간섭 | 일치 | 낮음 |

## 3. 상세 시나리오 분석

### 3.1. Binary Data Decoding (Buffer vs Uint8Array)
- **상황:** 서버(`msgpackr` on Node)는 바이너리 데이터를 `Buffer`로 디코딩하는 반면, 클라이언트(Browser)는 `Uint8Array`로 디코딩합니다.
- **분석:**
  - `Buffer`는 Node.js에서 `Uint8Array`의 하위 클래스처럼 동작합니다.
  - `calculateHash`는 객체의 키를 순회(`for..in`)하며 해시를 계산합니다.
  - 실험 결과, `Buffer`와 `Uint8Array` 모두 인덱스 키("0", "1", ...)만 순회하므로 해시 결과가 동일했습니다.
- **결론:** `Buffer` 사용으로 인한 해시 불일치는 발생하지 않습니다.

### 3.2. Floating Point Serialization (부동소수점 정밀도)
- **상황:** 메모리 상의 부동소수점 값(예: `0.1 + 0.2`)과 이를 JSON으로 저장 후 다시 로드했을 때의 값이 다를 수 있습니다.
- **분석:**
  - JavaScript(V8 엔진)는 IEEE 754 표준을 따르며, `JSON.stringify`와 `JSON.parse` 과정에서 정밀도 손실이 발생할 수 있는 극단적인 값들이 존재합니다.
  - 그러나 일반적인 사용 범위 내에서는 Node.js와 최신 브라우저 간의 동작이 일치했습니다.
  - `-0`과 `0`의 경우 해시 계산 로직(`node.toString()`)에서 동일하게 문자열 `"0"`으로 변환되어 일치했습니다.
- **결론:** 일반적인 수치 데이터에서는 문제가 없으나, 극단적인 정밀도가 요구되는 경우 잠재적 위험이 있습니다.

### 3.3. String Object vs Primitive String (주의 필요)
- **상황:** 클라이언트 코드에서 실수로 원시 문자열(`"text"`) 대신 래퍼 객체(`new String("text")`)를 생성하여 저장한 경우.
- **분석:**
  - `normalizeJSON`은 `typeof value !== 'object'`일 때만 값을 그대로 반환합니다.
  - `new String()`은 `typeof`가 `'object'`이므로, `normalizeJSON`은 이를 일반 객체로 취급하여 키("0", "1", "2", "3"...)를 순회합니다.
  - 반면 서버가 이를 JSON으로 로드할 때는 원시 문자열로 변환될 가능성이 높거나, 반대로 객체로 유지될 경우 해시 계산 방식이 완전히 달라집니다.
  - 실험 결과, `new String("test")`의 해시는 `0xa8545e97`인 반면, `"test"`의 해시는 `0xafd074ae`로 **불일치**했습니다.
- **결론:** 데이터 생성 시 래퍼 객체 사용을 지양해야 합니다.

### 3.4. JSON Normalization (Undefined Handling)
- **상황:** 객체 내에 `undefined` 값이 포함된 경우.
- **분석:**
  - `normalizeJSON`은 객체의 속성값이 `undefined`이면 해당 키를 결과 객체에 포함시키지 않습니다.
  - JSON 표준에서도 `undefined`는 직렬화되지 않습니다.
  - 따라서 클라이언트 메모리 상태(`{a: 1, b: undefined}`)와 서버의 JSON 로드 상태(`{a: 1}`)는 동일한 정규화 결과를 가지며 해시가 일치합니다.

## 4. 제언

실험 결과, RisuAI의 현재 해시 계산 로직은 대부분의 일반적인 시나리오(바이너리, 부동소수점, 희소 배열 등)에서 견고하게 동작합니다.

그러나 **"String Object"**와 같은 비표준 객체 사용이나, **외부 요인에 의한 데이터 변경**(예: 사용자가 파일 시스템을 직접 수정하여 JSON 포맷을 미세하게 변경) 시에는 해시 불일치가 발생할 수 있습니다.

따라서 앞서 적용한 **"Authoritative Hash Sync (서버 권위 해시 동기화)"** 방식은 이러한 잠재적, 미지의 불일치 시나리오를 모두 포괄하여 방어할 수 있는 가장 확실한 해결책입니다.
