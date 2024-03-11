---
description: 5장에서 언급한 안정 해시가 빛을 발하는 순간이다. Key-Value 저장소는 빠른 탐색과 RDBMS 부하 분산, 수평 확장에 용이하다.
---

# Chapter6. Key-Value 저장소 설계

단일 키-밸류 저장소는 만들기 쉽지만 금방 한계가 찾아온다. 데이터 압축이나 자주 활용되는 데이터만 메모리에 저장하는 방식으로 타이밍을 늦출 수는 있지만 한계가 명확하다.

### 분산 키-밸류 저장소

#### CAP 정리

*   cap 정리란, 일관성(`consistency`), 가용성(`availability`), 파티션 감내(`partition tolerance`) 라는 분산 시스템의 핵심 요소들을 모두 만족하는 설계란 불가능하다는 정리다. 간단하게 얘기해서, 완벽한 분산 설계란 불가능하다는 뜻

    <figure><img src="../.gitbook/assets/image.png" alt="" width="336"><figcaption></figcaption></figure>
*
