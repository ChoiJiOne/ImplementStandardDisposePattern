# Dispose 패턴

모든 프로그램은 실행 중에 메모리, 시스템 핸들 또는 데이터베이스 연결과 같은 하나 이상의 시스템 리소스를 획득하게 됩니다. 개발자는 이러한 시스템 리소스를 사용할 때 주의를 기울여야 하며, 리소스를 획득하고 사용한 후에는 반드시 해제해야 합니다.

CLR(Common Language Runtime)은 자동 메모리 관리를 지원합니다. C#의 `new` 연산자를 사용하여 할당된 관리 메모리는 명시적으로 해제할 필요가 없습니다. 가비지 컬렉터(GC)가 관리 메모리를 자동으로 해제하기 때문입니다. 이로 인해 개발자는 메모리 해제라는 번거롭고 어려운 작업에서 벗어나게 되며, 이는 .NET Framework가 제공하는 뛰어난 생산성의 주요 이유 중 하나입니다.

그러나 관리 메모리는 다양한 시스템 리소스 중 일부에 불과합니다. 관리 메모리가 아닌 리소스는 여전히 명시적으로 해제해야 하며, 이를 비관리 리소스라고 부릅니다. GC는 이러한 비관리 리소스를 관리하도록 설계되지 않았기 때문에, 비관리 리소스 관리의 책임은 전적으로 개발자에게 있습니다.

CLR은 비관리 리소스를 해제하는 데 도움이 되는 일부 기능을 제공합니다. `System.Object`는 `Finalize`라는 가상 메서드(종종 finalizer라고도 불립니다)를 선언합니다. 이 메서드는 GC가 객체의 메모리를 회수하기 전에 호출되며, 비관리 리소스를 해제하도록 오버라이드할 수 있습니다. finalizer를 오버라이드하는 타입은 finalizable 타입이라고 부릅니다.

finalizer는 일부 정리(cleanup) 시나리오에서 효과적일 수 있으나, 두 가지 중요한 단점이 있습니다:

- finalizer는 GC가 객체가 수집될 수 있다고 판단할 때 호출됩니다. 이는 리소스가 더 이상 필요하지 않게 된 시점 이후의 불확정한 시기에 발생합니다. 개발자가 리소스를 해제할 수 있거나 해제하고자 하는 시점과, 실제로 finalizer가 리소스를 해제하는 시점 사이에 발생하는 지연은, 많은 희소 리소스(쉽게 소진될 수 있는 리소스)를 획득하거나 사용 중인 리소스의 유지 비용이 큰 프로그램(예: 대규모 비관리 메모리 버퍼)에서는 허용되지 않을 수 있습니다.

- CLR이 finalizer를 호출해야 할 경우, finalizer가 실행되는 시점까지 해당 객체의 메모리 수집을 다음 가비지 컬렉션 라운드까지 미뤄야 합니다(finalizer는 컬렉션과 컬렉션 사이에서 실행됩니다). 이로 인해 해당 객체의 메모리(그리고 객체가 참조하는 모든 객체의 메모리)가 더 오랜 시간 동안 해제되지 않게 됩니다.

따라서 finalizer에만 의존하는 것은, 비관리 리소스를 최대한 빠르게 회수해야 할 때, 희소한 리소스를 다룰 때, 또는 finalization으로 인한 GC의 추가 오버헤드를 수용할 수 없는 고성능 시나리오에서 적합하지 않을 수 있습니다.

프레임워크는 `System.IDisposable` 인터페이스를 제공하며, 이는 개발자가 비관리 리소스를 더 이상 필요하지 않을 때 수동으로 해제할 수 있는 방법을 제공합니다. 또한 `GC.SuppressFinalize` 메서드를 통해 GC에 객체가 이미 수동으로 해제되었으므로 더 이상 finalizer가 필요하지 않다고 알려줄 수 있습니다. 이 경우, 객체의 메모리가 더 빨리 회수될 수 있습니다. `IDisposable`을 구현한 타입을 disposable 타입이라고 부릅니다.

Dispose 패턴은 finalizer와 `IDisposable` 인터페이스의 사용과 구현을 표준화하기 위해 고안되었습니다.

이 패턴의 주된 동기는 `Finalize` 및 `Dispose` 메서드의 구현 복잡성을 줄이는 데 있습니다. 이러한 복잡성은 두 메서드가 일부는 공유하고, 일부는 공유하지 않는 코드 경로를 가지기 때문입니다(이 장의 뒷부분에서 이러한 차이를 설명합니다). 또한, 언어에서 결정적 리소스 관리 지원이 발전해 온 과정에서 일부 패턴 요소가 역사적으로 자리 잡게 되었습니다.

✓ **반드시** disposable 타입의 인스턴스를 포함하는 타입에는 기본 Dispose 패턴을 구현해야 합니다. 기본 Dispose 패턴의 자세한 내용은 “기본 Dispose 패턴” 섹션을 참조하세요.

어떤 타입이 다른 disposable 객체의 수명도 함께 관리해야 할 경우, 이들을 함께 해제할 수 있어야 합니다. 이를 가능하게 하는 편리한 방법이 바로 컨테이너의 `Dispose` 메서드를 호출하는 것입니다.

✓ **반드시** 명시적으로 해제해야 하는 리소스를 보유하고 finalizer가 없는 타입에는 기본 Dispose 패턴을 구현하고 finalizer를 제공해야 합니다.

예를 들어, 비관리 메모리 버퍼를 저장하는 타입에는 이 패턴을 구현해야 합니다. finalizer 구현과 관련된 지침은 “Finalizable Types” 섹션에서 다룹니다.

✓ **고려할 것**: 자신은 비관리 리소스나 disposable 객체를 보유하지 않지만, 이를 보유할 가능성이 있는 서브타입이 있는 클래스에도 기본 Dispose 패턴을 구현하는 것이 좋습니다.

좋은 예는 `System.IO.Stream` 클래스입니다. 이 클래스는 추상 기반 클래스이므로 자체적으로는 리소스를 보유하지 않지만, 대부분의 서브클래스가 리소스를 보유하기 때문에 이 패턴을 구현하고 있습니다.

## Basic Dispose Pattern

이 패턴의 기본 구현은 `System.IDisposable` 인터페이스를 구현하고, `Dispose(bool)` 메서드를 선언하여 `Dispose` 메서드와 선택적으로 finalizer가 공유하는 모든 리소스 정리 로직을 구현하는 것입니다.

아래 예제는 기본 패턴의 간단한 구현을 보여줍니다:

```csharp
public class DisposableResourceHolder : IDisposable {

    private SafeHandle resource; // handle to a resource

    public DisposableResourceHolder() {
        this.resource = ... // allocates the resource
    }

    public void Dispose() {
        Dispose(true);
        GC.SuppressFinalize(this);
    }

    protected virtual void Dispose(bool disposing) {
        if (disposing) {
            if (resource!= null) resource.Dispose();
        }
    }
}
```

`disposing`이라는 이름의 불리언 매개변수는 메서드가 `IDisposable.Dispose` 구현에서 호출되었는지, 아니면 finalizer에서 호출되었는지를 나타냅니다. `Dispose(bool)` 구현에서는 이 매개변수를 확인한 후, 다른 참조 객체(예: 앞선 샘플의 `resource` 필드)를 참조해야 합니다. 이러한 객체들은 메서드가 `IDisposable.Dispose` 구현에서 호출될 때(즉, `disposing` 매개변수가 `true`일 때)만 접근해야 합니다. 만약 메서드가 finalizer에서 호출되면(`disposing`이 `false`일 때), 다른 객체를 참조해서는 안 됩니다. 그 이유는, 객체들이 예측할 수 없는 순서로 finalize되기 때문에, 해당 객체나 그 의존성이 이미 finalize되었을 수 있기 때문입니다.

또한, 이 내용은 Dispose 패턴을 이미 구현하지 않은 기반 클래스를 상속하는 클래스에 적용됩니다. 만약 상속받은 클래스가 이미 이 패턴을 구현하고 있다면, 단순히 `Dispose(bool)` 메서드를 오버라이드하여 추가적인 리소스 정리 로직을 구현하면 됩니다.

✓ **반드시** 비관리 리소스를 해제하는 모든 로직을 중앙화하기 위해 `protected virtual void Dispose(bool disposing)` 메서드를 선언해야 합니다.

모든 리소스 정리는 이 메서드에서 수행되어야 합니다. 이 메서드는 finalizer와 `IDisposable.Dispose` 메서드에서 모두 호출됩니다. finalizer 내부에서 호출될 경우 `disposing` 매개변수는 `false`가 됩니다. 이를 통해 finalization 중에는 다른 finalizable 객체에 접근하지 않도록 보장할 수 있습니다. finalizer 구현의 세부사항은 다음 섹션에서 다룰 예정입니다.

```csharp
protected virtual void Dispose(bool disposing) {
    if (disposing) {
        if (resource!= null) resource.Dispose();
    }
}
```

✓ **반드시** `IDisposable` 인터페이스를 구현할 때, 단순히 `Dispose(true)`를 호출하고 이어서 `GC.SuppressFinalize(this)`를 호출하도록 구현해야 합니다.

`SuppressFinalize` 호출은 `Dispose(true)`가 성공적으로 실행된 경우에만 수행되어야 합니다.

```csharp
public void Dispose(){
    Dispose(true);
    GC.SuppressFinalize(this);
}
```

X **절대** 매개변수가 없는 `Dispose` 메서드를 `virtual`로 선언하지 마세요.

`Dispose(bool)` 메서드만이 서브클래스에서 오버라이드되도록 설계되어야 합니다.

```csharp
// bad design
public class DisposableResourceHolder : IDisposable {
    public virtual void Dispose() { ... }
    protected virtual void Dispose(bool disposing) { ... }
}

// good design
public class DisposableResourceHolder : IDisposable {
    public void Dispose() { ... }
    protected virtual void Dispose(bool disposing) { ... }
}
```

X **절대** `Dispose()`와 `Dispose(bool)` 이외의 `Dispose` 메서드 오버로드를 선언하지 마세요.

`Dispose`는 이 패턴을 정립하고, 구현자, 사용자, 그리고 컴파일러 간의 혼란을 방지하기 위한 예약어처럼 간주됩니다. 일부 언어는 특정 타입에 대해 이 패턴을 자동으로 구현하도록 선택할 수도 있습니다.

✓ **반드시** `Dispose(bool)` 메서드가 여러 번 호출될 수 있도록 허용하세요. 이 메서드는 첫 번째 호출 이후에는 아무런 동작도 하지 않도록 설계할 수 있습니다.

```csharp
public class DisposableResourceHolder : IDisposable {

    bool disposed = false;

    protected virtual void Dispose(bool disposing) {
        if (disposed) return;
        // cleanup
        ...
        disposed = true;
    }
}
```

X **절대** `Dispose(bool)` 메서드 내에서 예외를 던지지 않도록 하세요. 단, 해당 프로세스가 손상되는 심각한 상황(리소스 누수, 불일치한 공유 상태 등)에서는 예외를 던질 수 있습니다.

사용자는 `Dispose` 호출이 예외를 발생시키지 않을 것이라 기대합니다.

만약 `Dispose`가 예외를 던질 수 있다면, 이후의 `finally` 블록에서 실행되어야 하는 정리 로직이 실행되지 않을 수 있습니다. 이를 피하기 위해 사용자는 모든 `Dispose` 호출(특히 `finally` 블록 안에서!)을 `try` 블록으로 감싸야 하며, 이는 매우 복잡한 정리 핸들러로 이어집니다. 또한, `Dispose(bool disposing)` 메서드를 실행할 때, `disposing`이 `false`일 경우에는 절대로 예외를 던지지 않도록 하세요. finalizer 컨텍스트 안에서 예외를 던지면 프로세스가 종료됩니다.

✓ **반드시** 이미 해제된 객체의 멤버를 호출할 경우 `ObjectDisposedException`을 던지도록 하세요.

```csharp
public class DisposableResourceHolder : IDisposable {
    bool disposed = false;
    SafeHandle resource; // handle to a resource

    public void DoSomething() {
        if (disposed) throw new ObjectDisposedException(...);
        // now call some native methods using the resource
        ...
    }
    protected virtual void Dispose(bool disposing) {
        if (disposed) return;
        // cleanup
        ...
        disposed = true;
    }
}
```

✓ **고려할 것**: 해당 분야에서 `close`가 표준 용어라면, `Dispose()` 외에도 `Close()` 메서드를 제공하는 것을 고려해 보세요.

이 경우, `Close` 구현을 `Dispose`와 동일하게 만들어야 하며, `IDisposable.Dispose` 메서드를 명시적으로 구현하는 것을 고려하는 것이 중요합니다.

```csharp
public class Stream : IDisposable {
    IDisposable.Dispose() {
        Close();
    }
    public void Close() {
        Dispose(true);
        GC.SuppressFinalize(this);
    }
}
```

## Finalizable Types

finalizable 타입은 finalizer를 오버라이드하고, `Dispose(bool)` 메서드에서 finalization 경로를 제공함으로써 기본 Dispose 패턴을 확장한 타입을 말합니다.

finalizer는 올바르게 구현하기가 매우 어렵습니다. 그 이유는 finalizer 실행 중에는 시스템 상태에 대해 일반적으로 유효한 가정을 할 수 없기 때문입니다. 따라서 다음 지침을 신중히 고려해야 합니다.

또한, 일부 지침은 `Finalize` 메서드뿐만 아니라 finalizer에서 호출되는 모든 코드에도 적용됩니다. 앞서 정의한 기본 Dispose 패턴의 경우, 이는 `disposing` 매개변수가 `false`인 상태에서 `Dispose(bool disposing)` 내부에서 실행되는 로직을 의미합니다.

만약 기반 클래스가 이미 finalizer를 구현하고, 기본 Dispose 패턴을 따르고 있다면, 다시 `Finalize`를 오버라이드하지 말아야 합니다. 대신, `Dispose(bool)` 메서드만 오버라이드하여 추가적인 리소스 정리 로직을 구현하면 됩니다.

아래 코드는 finalizable 타입의 예시를 보여줍니다:

```csharp
public class ComplexResourceHolder : IDisposable {

    private IntPtr buffer; // unmanaged memory buffer
    private SafeHandle resource; // disposable handle to a resource

    public ComplexResourceHolder() {
        this.buffer = ... // allocates memory
        this.resource = ... // allocates the resource
    }

    protected virtual void Dispose(bool disposing) {
        ReleaseBuffer(buffer); // release unmanaged memory
        if (disposing) { // release other disposable objects
            if (resource!= null) resource.Dispose();
        }
    }

    ~ComplexResourceHolder() {
        Dispose(false);
    }

    public void Dispose() {
        Dispose(true);
        GC.SuppressFinalize(this);
    }
}
```

X **절대** finalizable 타입을 만들지 않도록 하세요.

finalizer가 필요하다고 생각될 때는 반드시 신중히 검토해야 합니다. finalizer를 가진 인스턴스는 성능과 코드 복잡성 측면에서 실제 비용이 있습니다. 가능한 경우 `SafeHandle` 같은 리소스 래퍼를 사용하여 비관리 리소스를 캡슐화하는 것이 좋습니다. 이렇게 하면 finalizer는 필요하지 않으며, 래퍼가 자체적으로 리소스 정리를 담당합니다.

X **절대** 값 타입에 finalizer를 정의하지 마세요.

CLR은 참조 타입만 finalization을 수행하기 때문에, 값 타입에 finalizer를 정의하려고 해도 무시됩니다. C#과 C++ 컴파일러는 이 규칙을 강제합니다.

✓ **반드시** unmanaged 리소스를 해제하는 책임이 있고, 그 리소스가 자체 finalizer를 가지지 않은 타입이라면 finalizable 타입으로 구현해야 합니다.

finalizer를 구현할 때는 단순히 `Dispose(false)`를 호출하고, 모든 리소스 정리 로직은 `Dispose(bool disposing)` 메서드 안에 배치하세요.

```csharp
public class ComplexResourceHolder : IDisposable {

    ~ComplexResourceHolder() {
        Dispose(false);
    }

    protected virtual void Dispose(bool disposing) {
        ...
    }
}
```

✓ **반드시** finalizable 타입에서는 기본 Dispose 패턴을 구현해야 합니다.

이렇게 하면, 사용자가 finalizer가 담당하는 동일한 리소스를 명시적으로 해제할 수 있는 수단을 제공할 수 있습니다.

X **절대** finalizer 코드 경로에서는 다른 finalizable 객체에 접근하지 마세요. 이미 finalize되었을 가능성이 높기 때문입니다.

예를 들어, finalizable 객체 A가 finalizable 객체 B를 참조한다면, A의 finalizer에서 B를 안전하게 사용할 수 없습니다. finalizer들은 순서가 무작위로 호출되기 때문입니다(critical finalization의 약한 순서 보장 제외).

또한, 정적 변수에 저장된 객체들도 애플리케이션 도메인 언로드나 프로세스 종료 시점에 수집됩니다. 만약 `Environment.HasShutdownStarted`가 `true`를 반환한다면, finalizable 객체를 참조하는 정적 변수에 접근하거나 정적 메서드를 호출하는 것은 안전하지 않을 수 있습니다.

✓ **반드시** Finalize 메서드는 `protected`로 선언해야 합니다.

C#, C++, VB.NET 개발자는 컴파일러가 이 규칙을 자동으로 보장해 주므로 걱정할 필요가 없습니다.

X **절대** finalizer 로직에서 예외가 외부로 전파되지 않도록 하세요. 단, 시스템 치명적 오류의 경우는 예외입니다.

finalizer에서 예외가 발생하면, .NET Framework 2.0 이후부터 CLR은 전체 프로세스를 종료시킵니다. 이로 인해 다른 finalizer가 실행되지 않고, 리소스 해제가 통제되지 않는 방식으로 이루어질 수 있습니다.

✓ **고려할 것**: 강제로 애플리케이션 도메인 언로드나 스레드 중단이 발생해도 finalizer가 반드시 실행되어야 하는 상황이라면, `CriticalFinalizerObject`를 포함하는 타입 계층을 사용하여 “critical finalizable object”를 생성 및 사용하는 것을 고려해 보세요.











# 참조

- [Dispose Pattern- Framework Design Guidelines](https://learn.microsoft.com/ko-kr/dotnet/standard/design-guidelines/dispose-pattern)
