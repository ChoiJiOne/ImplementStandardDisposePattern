# 용어

| 용어                          | 설명                                                                               |
| --------------------------- | -------------------------------------------------------------------------------- |
| **관리되지 않는 리소스**             | .NET의 가비지 컬렉터가 추적할 수 없는 리소스(파일 핸들, 네트워크 연결, 윈도우 핸들 등). 직접 해제해야 함.                |
| **Dispose 패턴**              | `IDisposable.Dispose` 메서드와 선택적 finalizer를 함께 구현해, 명시적/비결정적 리소스 정리를 표준화하는 설계 패턴.  |
| **IDisposable 인터페이스**       | `Dispose()` 메서드를 선언하는 인터페이스로, 객체의 수명을 명확히 종료하기 위해 구현됨.                           |
| **Dispose(bool disposing)** | 관리 리소스와 비관리 리소스를 명확하게 구분해 해제할 수 있도록 하는 보호된 가상 메서드. true/false에 따라 호출 경로가 달라짐.    |
| **파이널라이저(Finalizer)**       | `Object.Finalize()`를 오버라이드해, `Dispose`가 호출되지 않은 경우에도 비관리 리소스를 비결정적으로 해제할 수 있게 함. |
| **GC.SuppressFinalize**     | `Dispose`가 호출되었음을 GC에 알려, 파이널라이저가 불필요하게 호출되지 않도록 억제함.                            |
| **SafeHandle**              | 비관리 리소스를 감싸서 파이널라이저에서 안전하게 해제할 수 있도록 하는 .NET의 추상 클래스. 직접 파이널라이저 구현을 대신할 수 있음.    |
| **멱등성(idempotent)**         | `Dispose`가 여러 번 호출되더라도 안전하게 동작해야 함(같은 해제를 중복해서 수행하면 안 됨).                        |
| **finalizable 타입**          | `Finalize` 메서드를 오버라이드한 타입으로, 파이널라이저 경로를 가지는 타입.                                  |
| **CriticalFinalizerObject** | 중요한 리소스 정리를 위한 critical finalization을 보장하는 .NET 타입 계층의 기반 클래스.                   |
