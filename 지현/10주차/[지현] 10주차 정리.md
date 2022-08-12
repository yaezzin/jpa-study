# 1. JPA 예외처리

### JPA 표준 예외

1. JPA 표준 예외는 모두 PersistenceException의 자식클래스, PersistenceException는 RunTimeException의 자식클래스
   <img width=300 src="https://user-images.githubusercontent.com/87467801/184294009-c8cfdb94-5cf6-469a-b72d-0247a429f7ab.png">    
2. RunTimeException을 상속받기 때문에 PersistenceException는 모두 명시적인 예외처리를 강제하지 않는 언체크 예외(코드에서 미리 조건을 체크하도록 만들면 개발자가 충분히 피할 수 있는 예외들)
3. 트랜잭션 롤백을 표시하는 예외
    - 복구하면 안되는 심각한 예외
    - 강제로 commit하면 RollBackException 발생
    - e.g. EntityExistsException, EntityNotFoundException, OptimisticLockException, PessimisticLockException, RollbackException, TransactionRequiredException
4. 트랜잭션 롤백을 표시하지 않는 예외
    - 개발자가 롤백 여부를 판단 가능
    - e.g. NoResultException, NonUniqueResultException, LockTimeoutException, QueryTimeoutException

### 스프링 프레임워크의 JPA 예외 변환

1. 서비스 계층에서 JPA 예외를 직접 사용하면 서비스 계층이 데이터  접근 계층의 구현 기술에 의존하게 됨
2. 스프링 프레임워크가 JPA 표준 예외를 추상화해서 개발자에게는 스프링 예외를 제공 
→ 서비스 계층은 JPA에 의존적인 예외를 처리하지 않아도 됨
3. JPA 예외 변환기 (PersistenceExceptionTranslationPostProcessor) → appConfig.xml 또는 JavaConfig에 빈으로 등록 → **스프링 부트 최신 버전은 자동 등록. 따로 등록할 필요X**
4. `@Repository`를 사용한 곳에 예외 변환 AOP 사용하여 스프링 추상 예외로 변환
    
    ```java
    @Resository
    public class NoResultExceptionTestRepository {
    
    	@PersistenceContext EntityManager em;
    
    	public Member findMember() {
    
    		// 조회 데이터가 없을 때
    		return em.creatQuery("select m from Member m", Member.class).getSinlgeResult();
    	}
    }
    
    // PersistenceExceptionTranslationPostProcessor가 등록한 AOP 인터셉터 동작
    // javax.persistence.NoResultException -> org.springframework.dao.EmptyResultDataAccessException
    ```
    
5. 예외를 변환하지 않는 예시
    
    ```java
    @Repository
    public class NoResultExceptionTestRepository {
    
    	@PersistenceContext EntityManager em;
    
    	public Member findMember() thorws javax.persistence.NoResultException {
    		return em.creatQuery("select m from Member m", Member.class).getSinlgeResult();
    	}
    }
    ```
    

### 트랜잭션 롤백 시 주의사항

1. 트랜잭션을 롤백하는 것은 DB의 반영사항만 롤백하고 수정한 자바 객체는 복구하지 않고 영속성 컨텍스트에 남아 있음 → 롤백 후에는 영속성 컨텍스트를 새로 생성하거나 초기화(`EntityManager.clear()`) 후 사용
2. 스프링 프레임워크의 문제 해결 방법
    - 트랜잭션당 영속성 컨텍스트 전략(기본 전략)
        - 트랜잭션 AOP 종료 시점에 트랜잭션을 롤백하면서 영속성 컨텍스트도 함께 종료
    - 여러 트랜잭션이 하나의 영속성 컨텍스트를 사용할 때 (e.g. OSIV)
        - 영속성 컨텍스트의 범위를 트랜잭션 범위보다 넓게 사용할 때는 트랜잭션을 롤백해서 영속성 컨텍스트에 이상이 생겨도 다른 트랜잭션에서 해당 영속성 컨텍스트를 그대로 사용하는 문제
        - 스프링 프레임워크가 트랜잭션 롤백시 영속성 컨텍스트를 자동으로 초기화
        
        ```java
        @Override
        	protected void doRollback(DefaultTransactionStatus status) {
        		JpaTransactionObject txObject = (JpaTransactionObject) status.getTransaction();
        		if (status.isDebug()) {
        			logger.debug("Rolling back JPA transaction on EntityManager [" +
        					txObject.getEntityManagerHolder().getEntityManager() + "]");
        		}
        		try {
        			EntityTransaction tx = txObject.getEntityManagerHolder().getEntityManager().getTransaction();
        			if (tx.isActive()) {
        				tx.rollback();
        			}
        		}
        		catch (PersistenceException ex) {
        			throw new TransactionSystemException("Could not roll back JPA transaction", ex);
        		}
        		finally {
        			if (!txObject.isNewEntityManagerHolder()) {
        				// Clear all pending inserts/updates/deletes in the EntityManager.
        				// Necessary for pre-bound EntityManagers, to avoid inconsistent state.
        				txObject.getEntityManagerHolder().getEntityManager().clear();
        			}
        		}
        	}
        ```
