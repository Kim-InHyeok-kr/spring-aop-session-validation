# AOP 기반 사용자 세션 검증 로직 구현 사례

## 1. AOP와 Filter 개념 정리

### AOP (Aspect Oriented Programming)
- AOP는 공통 관심사(Cross-Cutting Concern)를 모듈화하여 핵심 비즈니스 로직과 분리하는 프로그래밍 패러다임입니다.
- 예: 로깅, 트랜잭션 관리, 보안 검사, 권한 체크 등을 비즈니스 로직과 분리하여 유지보수를 용이하게 합니다.
- Spring에서는 `@Aspect`, `@Around`, `@Before`, `@After` 등을 통해 적용합니다.

### Filter
- Filter는 Servlet 기반 Web Application에서 HTTP 요청/응답 전후로 공통 작업을 처리할 수 있는 컴포넌트입니다.
- 예: 인증, 인코딩 처리, CORS 설정 등을 위한 사전/사후 처리에 주로 사용합니다.
- 요청(Request)과 응답(Response) 단위로 동작하며, Web 컨텍스트 레벨에서 작동합니다.

### AOP와 Filter 차이 요약
| 항목 | AOP | Filter |
|:---|:---|:---|
| 동작 레벨 | Spring Bean 메서드 레벨 | HTTP 요청/응답 레벨 |
| 주요 목적 | 로직 모듈화 (관심사 분리) | 요청 전/후 공통 처리 |
| 사용 시점 | 메서드 호출 전후 | 요청(Request) 수신 전후 |
| 적용 범위 | 특정 Bean, 특정 메서드 단위 | 전역 요청 처리 |

---

## 2. 프로젝트에서 AOP를 선택한 이유

본 사례에서는 **전자결재**, **게시판** 등의 기능에서 사용자 세션 데이터 기반 권한 검증을 수행해야 했습니다. 이 검증 로직은 특정 컨트롤러(`EmpController`)의 특정 API 메서드 앞에서만 공통으로 적용되어야 했기 때문에, HTTP 요청 전체를 대상으로 동작하는 Filter 대신, **AOP를 사용**하여 보다 세밀하고 유연하게 제어할 수 있었습니다.

**선택 이유 요약:**
- 컨트롤러 단 메서드 레벨에서만 공통 로직을 적용할 필요
- 세션 기반 검증이 비즈니스 로직과 밀접하게 연관되어 있음
- 핵심 비즈니스 로직과 분리하여 유지보수성 확보
- HTTP 요청 전체가 아닌 특정 컨트롤러/메서드만 타겟팅

또한 이 구조를 통해 **ISO27001 보안 인증** 요건(사용자 세션 데이터 기반 인증 및 권한 검증)을 충족할 수 있었습니다.

---

## 3. 구현 소스 예제

```java
@Around("execution(* com.vntg.app.controller.api.emp.EmpController.EmpStatusMeWorkModi(..)) || " +
        "execution(* com.vntg.app.controller.api.emp.EmpController.setEmpStatusScheduleRegInfo(..)) || " +
        "execution(* com.vntg.app.controller.api.msg.MsgController.getSMSInfo(..))")
public Object sesssionCheck(JoinPoint joinPoint) throws Throwable {

  String methodName = joinPoint.getSignature().getName(); //메소드 명
  Object[] args = joinPoint.getArgs();

  Map<String, Object> params = apiComm.setAopSessionData(args); // 세선 데이터

  logger.info("SessionCheckAop :: sessionCheck() - methodName:{}, params:{}", methodName, params);

  try {
    switch (methodName) {
    case "EmpStatusMeWorkModi":
      sessionCheckService.EmpStatusMeSessionCheck(params);
      break;
    case "setEmpStatusScheduleRegInfo":
      sessionCheckService.EmpStatusPlanSessionCheck(params);
      break;
    case "getSMSInfo":
      sessionCheckService.SMSSessionCheck(params);
      break;
    }
    
  } catch (Exception e) {
    logger.error("SessionCheckAop :: sessionCheck - error : {}", e);
    throw new Exception(APIResCode.CD_ERROR);
  }
  
  // 타겟 메서드 실행
  Object result = ((ProceedingJoinPoint) joinPoint).proceed();

  return result;
}
```

### 주요 포인트
- `@Around`를 사용하여 `EmpController` 하위 모든 메서드를 감지
- 호출 메서드명(`apiAddress`)에 따라 권한 검증 방식 분기 처리
- 검증 실패 시 예외 발생 → 에러 로깅 및 API 표준 응답 코드 반환
- 검증 통과 시 원래 비즈니스 로직 실행

---

## 4. 정리

본 AOP 적용 사례를 통해 다음을 달성할 수 있었습니다:

- ✅ 사용자 인증/권한 검증의 일관성과 보안성 강화
- ✅ 비즈니스 코드와 보안 검증 코드 분리 (관심사 분리)
- ✅ ISO27001 보안 인증 요건 충족
- ✅ 유지보수성과 확장성 향상 (컨트롤러 메서드 추가 시 AOP 로직만 수정하면 됨)

AOP를 적절히 활용하면 시스템의 복잡도를 줄이고, 공통 기능 관리 효율성을 극대화할 수 있습니다.

