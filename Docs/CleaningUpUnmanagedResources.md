# 관리되지 않은 리소스 정리

애플리케이션에서 생성하는 대부분의 객체에 대해서는 .NET의 가비지 컬렉터가 메모리 관리를 처리하도록 맡길 수 있습니다. 그러나 관리되지 않는 리소스를 포함하는 객체를 생성할 경우, 사용을 마친 후에 이러한 리소스를 명시적으로 해제해야 합니다. 가장 일반적인 관리되지 않는 리소스는 파일, 창, 네트워크 연결 또는 데이터베이스 연결과 같은 운영 체제 리소스를 래핑하는 객체들입니다. 가비지 컬렉터는 관리되지 않는 리소스를 캡슐화하는 객체의 수명은 추적할 수 있지만, 이러한 관리되지 않는 리소스를 해제하고 정리하는 방법은 알지 못합니다.

객체가 관리되지 않는 리소스를 사용하는 경우, 다음 사항을 수행해야 합니다:

- Dispose 패턴을 구현하세요. 이를 위해 `IDisposable.Dispose`를 구현하여 관리되지 않는 리소스를 명확하게 해제할 수 있게 해야 합니다. 타입의 사용자가 해당 객체(그리고 그 객체가 사용하는 리소스)가 더 이상 필요하지 않을 때 `Dispose`를 호출합니다. `Dispose` 메서드는 관리되지 않는 리소스를 즉시 해제합니다.

- 타입의 사용자가 `Dispose` 호출을 깜빡했을 경우, 관리되지 않는 리소스를 해제할 수 있는 방법을 제공해야 합니다. 이 방법에는 두 가지가 있습니다:
  
  - 안전 핸들을 사용하여 관리되지 않는 리소스를 래핑하는 것입니다. 이 방법이 권장됩니다. 안전 핸들은 `System.Runtime.InteropServices.SafeHandle` 추상 클래스에서 파생되며, 견고한 `Finalize` 메서드를 포함하고 있습니다. 안전 핸들을 사용할 경우, 단순히 `IDisposable` 인터페이스를 구현하고, `IDisposable.Dispose` 구현 내에서 안전 핸들의 `Dispose` 메서드를 호출하면 됩니다. 안전 핸들의 `Dispose`가 호출되지 않으면, 가비지 컬렉터가 안전 핸들의 파이널라이저를 자동으로 호출합니다.
  - 파이널라이저를 정의하세요. 파이널라이저는 타입의 사용자가 `IDisposable.Dispose`를 호출하지 않아도 관리되지 않는 리소스를 비결정적으로 해제할 수 있게 합니다.

> ⚠️ 경고
> 
> 객체 파이널라이제이션은 복잡하고 오류를 일으키기 쉬운 작업이므로, 자체 파이널라이저를 제공하기보다는 안전 핸들을 사용하는 것을 권장합니다.

타입의 사용자는 관리되지 않는 리소스가 사용하는 메모리를 해제하기 위해, `IDisposable.Dispose` 구현을 직접 호출할 수 있습니다. `Dispose` 메서드를 올바르게 구현하면, `Dispose` 메서드가 호출되지 않은 경우에도 안전 핸들의 `Finalize` 메서드나 `Object.Finalize`의 오버라이드가 리소스를 정리하기 위한 안전망 역할을 수행합니다.

## Dispose 메서드 구현

Dispose 메서드는 주로 관리되지 않는 리소스를 해제하기 위해 구현됩니다. IDisposable을 구현하는 인스턴스 멤버를 사용할 때는, 일반적으로 Dispose 호출을 연쇄적으로 연결해서 처리합니다. Dispose를 구현하는 다른 이유로는, 할당된 메모리를 해제하거나, 컬렉션에 추가된 항목을 제거하거나, 획득한 잠금의 해제를 신호로 표시하는 경우 등이 있습니다.

.NET의 가비지 컬렉터는 관리되지 않는 메모리를 할당하거나 해제하지 않습니다. 객체를 소멸시키는 패턴, 즉 Dispose 패턴은 객체의 수명 주기를 체계적으로 관리하기 위한 것입니다. Dispose 패턴은 IDisposable 인터페이스를 구현하는 객체에서 사용됩니다. 파일 및 파이프 핸들, 레지스트리 핸들, 대기 핸들 또는 관리되지 않는 메모리 블록에 대한 포인터와 상호 작용할 때 일반적으로 이 패턴을 사용합니다. 이는 가비지 컬렉터가 관리되지 않는 개체를 회수할 수 없기 때문입니다.

리소스가 항상 적절하게 정리되도록 보장하기 위해, Dispose 메서드는 멱등성(idempotent)을 가져야 합니다. 즉, 여러 번 호출되어도 예외를 발생시키지 않아야 하며, 이후의 Dispose 호출은 아무 작업도 수행하지 않아야 합니다.

GC.KeepAlive 메서드 예제는 개체나 그 멤버에 대한 관리되지 않는 참조가 여전히 사용 중인 동안, 가비지 컬렉션으로 인해 종료자가 호출될 수 있음을 보여줍니다. 현재 루틴의 시작부터 이 메서드가 호출되는 시점까지 개체가 가비지 컬렉션의 대상이 되지 않도록 하려면, GC.KeepAlive를 활용하는 것이 합리적일 수 있습니다.

> 💡 팁
> 
> 의존성 주입과 관련하여, `IServiceCollection`에 서비스를 등록할 때는 서비스의 수명이 자동으로 관리됩니다. `IServiceProvider`와 해당 `IHost`가 리소스 정리를 조율합니다. 특히, `IDisposable` 및 `IAsyncDisposable`을 구현하는 경우, 이들이 명시된 수명이 끝나면 적절하게 해제됩니다.
> 
> 자세한 내용은 [.NET의 의존성 주입 문서]([Dependency injection - .NET | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection))를 참고하세요.

### 계단식 처리 해제 호출

클래스가 IDisposable을 구현하는 다른 타입의 인스턴스를 소유하고 있다면, 그 포함하는 클래스 자체도 IDisposable을 구현해야 합니다. 일반적으로 IDisposable 구현체를 인스턴스 멤버(또는 프로퍼티)로 보유하는 클래스는, 이 멤버의 정리 작업도 책임집니다. 이를 통해 참조된 IDisposable 타입이 Dispose 메서드를 통해 명확하게 정리 작업을 수행할 기회를 가질 수 있습니다. 아래 예제에서 클래스는 `sealed`(또는 Visual Basic에서는 `NotInheritable`)로 선언됩니다.

```csharp
using System;

public sealed class Foo : IDisposable
{
    private readonly IDisposable _bar;

    public Foo()
    {
        _bar = new Bar();
    }

    public void Dispose() => _bar.Dispose();
}
```

> 💡 팁
> 
> 클래스에 `IDisposable` 필드나 프로퍼티가 있더라도, 해당 리소스의 소유권이 없다면 `IDisposable`을 구현할 필요는 없습니다. 일반적으로 `IDisposable` 객체를 생성하고 보관하는 클래스가 소유권을 갖지만, 경우에 따라 이 소유권이 다른 `IDisposable` 타입으로 이전될 수 있습니다.
> 
> 또한, 파이널라이저(또는 파이널라이저에 의해 호출되는 `Dispose(false)` 메서드)에서 null 확인을 수행해야 할 때가 있습니다. 주된 이유 중 하나는 인스턴스가 완전히 초기화되지 않았을 수 있기 때문입니다(예: 생성자에서 예외가 발생한 경우).



### Dispose() 와 Dispose(bool)

`IDisposable` 인터페이스는 매개변수가 없는 단일 메서드 `Dispose`를 구현하도록 요구합니다. 또한, `sealed`가 아닌 클래스라면 `Dispose(bool)` 오버로드 메서드를 반드시 제공해야 합니다.

메서드 시그니처는 다음과 같습니다:

- `public` 비가상 메서드(`NotOverridable` in Visual Basic): `IDisposable.Dispose` 구현.

- `protected` 가상 메서드(`Overridable` in Visual Basic): `Dispose(bool)`.

#### Dispose() method

공개적이고 비가상(Visual Basic의 경우 `NotOverridable`)인 매개변수가 없는 `Dispose` 메서드는, 타입의 소비자가 더 이상 필요하지 않을 때 호출됩니다. 이 메서드의 목적은 관리되지 않는 리소스를 해제하고, 일반적인 정리 작업을 수행하며, 파이널라이저가 존재한다면 그것이 실행될 필요가 없음을 나타내는 것입니다. 관리되는 객체에 연결된 실제 메모리 해제는 항상 가비지 컬렉터의 역할입니다. 이러한 이유로, 이 메서드는 표준 구현 방식을 따릅니다:

```csharp
public void Dispose()
{
    // Dispose of unmanaged resources.
    Dispose(true);
    // Suppress finalization.
    GC.SuppressFinalize(this);
}
```

`Dispose` 메서드는 객체의 모든 정리 작업을 수행하므로, 가비지 컬렉터는 객체의 `Object.Finalize` 오버라이드를 더 이상 호출할 필요가 없습니다. 따라서 `SuppressFinalize` 메서드를 호출하면 가비지 컬렉터가 파이널라이저를 실행하지 않도록 방지합니다. 타입에 파이널라이저가 없다면, `GC.SuppressFinalize` 호출은 아무런 효과도 없습니다. 실제 정리 작업은 `Dispose(bool)` 메서드 오버로드가 수행합니다.

#### Dispose(bool) method overload

이 오버로드에서 `disposing` 매개변수는 호출이 `Dispose` 메서드에서 온 것인지(값이 `true`) 또는 파이널라이저에서 온 것인지(값이 `false`)를 나타내는 불리언 값입니다.

```csharp
protected virtual void Dispose(bool disposing)
{
    if (_disposed)
    {
        return;
    }

    if (disposing)
    {
        // Dispose managed state (managed objects).
        // ...
    }

    // Free unmanaged resources.
    // ...

    _disposed = true;
}
```

> ⚠️ 중요
> 
> `disposing` 매개변수는 파이널라이저에서 호출될 때는 `false`여야 하고, `IDisposable.Dispose` 메서드에서 호출될 때는 `true`여야 합니다. 다시 말해, 명시적으로 호출될 때는 `true`, 비결정적으로 호출될 때는 `false`입니다.

메서드 본문은 세 개의 코드 블록으로 구성됩니다:

- 객체가 이미 해제된 경우 조건적으로 반환하는 블록.

- 관리되는 리소스를 해제하는 조건부 블록. 이 블록은 `disposing` 값이 `true`일 때 실행됩니다. 여기서 해제할 수 있는 관리되는 리소스에는 다음이 포함됩니다:
  
  - `IDisposable`을 구현한 관리되는 객체. 이 조건부 블록을 통해 이들의 `Dispose` 구현을 호출할 수 있습니다(연쇄 Dispose). 관리되지 않는 리소스를 `System.Runtime.InteropServices.SafeHandle`의 파생 클래스로 래핑한 경우, 이 자리에서 `SafeHandle.Dispose()`를 호출해야 합니다.
  
  - 대량의 메모리를 사용하거나 제한된 리소스를 사용하는 관리되는 객체. 큰 관리 객체 참조를 `null`로 설정하여 도달 불가능 상태로 만드는 것이 일반적입니다. 이렇게 하면 비결정적으로 회수되는 것보다 빠르게 해제됩니다.

- 관리되지 않는 리소스를 해제하는 블록. 이 블록은 `disposing` 매개변수의 값과 관계없이 항상 실행됩니다.

메서드 호출이 파이널라이저에서 온 경우, 관리되지 않는 리소스를 해제하는 코드만 실행되어야 합니다. 구현자는 `false` 경로에서 이미 해제된 관리 객체와 상호작용하지 않도록 주의해야 합니다. 이는 가비지 컬렉터가 파이널라이제이션 중 관리 객체를 해제하는 순서가 비결정적이기 때문입니다.

## Dispose 패턴 구현

`sealed`가 아닌 모든 클래스(또는 Visual Basic에서 `NotInheritable`로 지정되지 않은 클래스)는 상속될 가능성이 있는 기본 클래스 후보로 간주됩니다. 만약 이러한 클래스에서 디스포즈 패턴을 구현한다면, 다음 메서드들을 반드시 클래스에 포함해야 합니다:

- `Dispose(bool)` 메서드를 호출하는 `Dispose` 구현.

- 실제 정리 작업을 수행하는 `Dispose(bool)` 메서드.

- 관리되지 않는 리소스를 다루는 경우, `Object.Finalize` 메서드를 오버라이드하거나 관리되지 않는 리소스를 `SafeHandle`로 래핑.

이렇게 하면, 파생 클래스도 안전하고 정확하게 정리 작업을 수행할 수 있는 기반을 갖게 됩니다.

> 파이널라이저(즉, `Object.Finalize` 오버라이드)는 관리되지 않는 리소스를 직접 참조할 때만 필요합니다. 이 시나리오는 고급 시나리오로, 보통은 피할 수 있습니다:
> 
> - 클래스가 관리되는 객체만 참조한다면, 디스포즈 패턴을 구현하더라도 파이널라이저를 구현할 필요는 없습니다.
> 
> - 관리되지 않는 리소스를 다뤄야 한다면, 관리되지 않는 `IntPtr` 핸들을 `SafeHandle`로 래핑하는 것을 강력히 권장합니다. `SafeHandle`은 자체적으로 파이널라이저를 제공하므로, 직접 작성할 필요가 없습니다. 자세한 내용은 Safe handles 항목을 참고하세요.

### 관리되는 리소스를 가진 Base 클래스

다음은 관리되는 리소스만 소유하는 기본 클래스에서 디스포즈 패턴을 구현하는 일반적인 예제입니다.

```csharp
using System;
using System.IO;

public class DisposableBase : IDisposable
{
    // Detect redundant Dispose() calls.
    private bool _isDisposed;

    // Instantiate a disposable object owned by this class.
    private Stream? _managedResource = new MemoryStream();

    // Public implementation of Dispose pattern callable by consumers.
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }

    // Protected implementation of Dispose pattern.
    protected virtual void Dispose(bool disposing)
    {
        if (!_isDisposed)
        {
            _isDisposed = true;

            if (disposing)
            {
                // Dispose managed state.
                _managedResource?.Dispose();
                _managedResource = null;
            }
        }
    }
}
```

> 💡 참고
> 
> 이전 예제는 디스포즈 패턴을 설명하기 위해 가상의 `MemoryStream` 객체를 사용했습니다. 대신 어떤 `IDisposable` 객체라도 사용될 수 있습니다.

### 관리되지 않는 리소스를 가진 기본 클래스

다음은 `Object.Finalize`를 오버라이드하여 관리되지 않는 리소스를 정리하는 기본 클래스의 디스포즈 패턴 구현 예제입니다. 이 예제는 또한 `Dispose(bool)`를 스레드 안전하게 구현하는 방법을 보여줍니다. 다중 스레드 애플리케이션에서 관리되지 않는 리소스를 다룰 때 동기화는 중요할 수 있습니다. 앞서 언급했듯, 이 시나리오는 고급 시나리오입니다.

```csharp
using System;
using System.IO;
using System.Runtime.InteropServices;
using System.Threading;

public class DisposableBaseWithFinalizer : IDisposable
{
    // Detect redundant Dispose() calls in a thread-safe manner.
    // _isDisposed == 0 means Dispose(bool) has not been called yet.
    // _isDisposed == 1 means Dispose(bool) has been already called.
    private int _isDisposed;

    // Instantiate a disposable object owned by this class.
    private Stream? _managedResource = new MemoryStream();

    // A pointer to 10 bytes allocated on the unmanaged heap.
    private IntPtr _unmanagedResource = Marshal.AllocHGlobal(10);

    ~DisposableBaseWithFinalizer() => Dispose(false);

    // Public implementation of Dispose pattern callable by consumers.
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }

    // Protected implementation of Dispose pattern.
    protected virtual void Dispose(bool disposing)
    {
        // In case _isDisposed is 0, atomically set it to 1.
        // Enter the branch only if the original value is 0.
        if (Interlocked.CompareExchange(ref _isDisposed, 1, 0) == 0)
        {
            if (disposing)
            {
                _managedResource?.Dispose();
                _managedResource = null;
            }

            Marshal.FreeHGlobal(_unmanagedResource);
        }
    }
}
```

> 💡 참고
> 
> - 이전 예제에서는 `AllocHGlobal`을 사용하여 생성자에서 관리되지 않는 힙에 10바이트를 할당하고, `Dispose(bool)`에서 `FreeHGlobal`을 호출하여 버퍼를 해제했습니다. 이 예제는 단순히 설명을 위한 가상의 할당입니다.  
> 
> - 다시 말하지만, 파이널라이저를 구현하지 않는 것이 좋습니다. 앞선 예제의 동등한 기능을 안전하게 수행하고, 비결정적 정리와 동기화를 `SafeHandle`에 위임하는 방식을 보려면 “사용자 정의 안전 핸들을 사용한 디스포즈 패턴 구현”을 참고하세요.



> 💡 팁
> 
> C#에서 파이널라이제이션은 `Object.Finalize`를 오버라이드하는 방식이 아니라, 파이널라이저(소멸자)를 정의하여 구현됩니다.  
> Visual Basic에서는 `Protected Overrides Sub Finalize()`를 사용하여 파이널라이저를 만듭니다.

### 파생 클래스에서 Dispose 패턴 구현하기

`IDisposable` 인터페이스를 구현하는 기본 클래스로부터 파생된 클래스는, `IDisposable`을 다시 구현할 필요가 없습니다. 기본 클래스의 `IDisposable.Dispose` 구현이 파생 클래스에 상속되기 때문입니다. 대신, 파생 클래스의 정리 작업을 수행하기 위해 다음을 제공해야 합니다:

- 기본 클래스 메서드를 오버라이드하는 `protected override void Dispose(bool)` 메서드. 이 메서드는 파생 클래스의 실제 정리 작업을 수행하며, 반드시 `base.Dispose(bool)`(Visual Basic에서는 `MyBase.Dispose(bool)`)을 호출하고 `disposing` 상태를 전달해야 합니다.

- 관리되지 않는 리소스를 래핑하는 `SafeHandle`에서 파생된 클래스(권장) 또는 `Object.Finalize` 메서드를 오버라이드한 구현. `SafeHandle` 클래스는 파이널라이저를 제공하므로, 이를 사용하면 파이널라이저를 직접 작성할 필요가 없습니다. 만약 파이널라이저를 제공해야 한다면, 반드시 `Dispose(bool)`를 `false` 인자로 호출해야 합니다.

다음은 `SafeHandle`을 사용하는 파생 클래스의 디스포즈 패턴 구현의 일반적인 예제입니다:

```csharp
using System.IO;

public class DisposableDerived : DisposableBase
{
    // To detect redundant calls
    private bool _isDisposed;

    // Instantiate a disposable object owned by this class.
    private Stream? _managedResource = new MemoryStream();

    // Protected implementation of Dispose pattern.
    protected override void Dispose(bool disposing)
    {
        if (!_isDisposed)
        {
            _isDisposed = true;

            if (disposing)
            {
                _managedResource?.Dispose();
                _managedResource = null;
            }
        }

        // Call base class implementation.
        base.Dispose(disposing);
    }
}
```

> 💡 참고
> 
> 이전 예제에서는 디스포즈 패턴을 설명하기 위해 `SafeFileHandle` 객체를 사용했습니다. 그러나 `SafeHandle`에서 파생된 다른 객체도 사용할 수 있습니다. 예제는 `SafeFileHandle` 객체를 적절히 인스턴스화하지는 않았다는 점에 주의하세요.

다음은 `Object.Finalize`를 오버라이드하는 파생 클래스에서 디스포즈 패턴을 구현하는 일반적인 패턴입니다:

```csharp
using System.Threading;

public class DisposableDerivedWithFinalizer : DisposableBaseWithFinalizer
{
    // Detect redundant Dispose() calls in a thread-safe manner.
    // _isDisposed == 0 means Dispose(bool) has not been called yet.
    // _isDisposed == 1 means Dispose(bool) has been already called.
    private int _isDisposed;

    ~DisposableDerivedWithFinalizer() => Dispose(false);

    // Protected implementation of Dispose pattern.
    protected override void Dispose(bool disposing)
    {
        // In case _isDisposed is 0, atomically set it to 1.
        // Enter the branch only if the original value is 0.
        if (Interlocked.CompareExchange(ref _isDisposed, 1, 0) == 0)
        {
            if (disposing)
            {
                // TODO: dispose managed state (managed objects).
            }

            // TODO: free unmanaged resources (unmanaged objects) and override a finalizer below.
            // TODO: set large fields to null.
        }

        // Call the base class implementation.
        base.Dispose(disposing);
    }
}
```

## Safe handles

객체의 파이널라이저를 직접 작성하는 것은 복잡한 작업이며, 올바르게 구현되지 않으면 문제를 일으킬 수 있습니다. 따라서 파이널라이저를 구현하는 대신 `System.Runtime.InteropServices.SafeHandle` 객체를 사용하는 것을 권장합니다.

`System.Runtime.InteropServices.SafeHandle`은 관리되지 않는 리소스를 식별하는 `System.IntPtr`을 래핑하는 추상 관리형 타입입니다. Windows에서는 핸들, Unix에서는 파일 디스크립터를 식별할 수 있습니다. `SafeHandle`은 이 리소스가 한 번만 정확히 해제되도록 보장하는 모든 로직을 제공합니다. `SafeHandle`이 Dispose될 때, 혹은 `SafeHandle`의 모든 참조가 해제되어 `SafeHandle` 인스턴스가 파이널라이즈될 때 이 리소스는 해제됩니다.

`System.Runtime.InteropServices.SafeHandle`은 추상 기본 클래스입니다. 파생 클래스는 다양한 종류의 핸들을 다루기 위한 구체적 인스턴스를 제공합니다. 이러한 파생 클래스는 어떤 `System.IntPtr` 값이 유효하지 않은지를 검증하고, 핸들을 실제로 어떻게 해제할지를 정의합니다. 예를 들어, `SafeFileHandle`은 `SafeHandle`에서 파생되어 열려 있는 파일 핸들과 디스크립터를 식별하는 `IntPtr`을 래핑하고, Unix에서는 `close` 함수, Windows에서는 `CloseHandle` 함수를 통해 닫도록 `SafeHandle.ReleaseHandle()`을 오버라이드합니다. .NET 라이브러리의 대부분의 API는 관리되지 않는 리소스를 생성할 때 이를 `SafeHandle`로 래핑해서 반환해 주며, 원시 포인터를 직접 반환하지는 않습니다. 관리되지 않는 구성 요소와 상호작용하고 관리되지 않는 리소스의 `IntPtr`을 얻은 경우, 이를 래핑하는 자체 `SafeHandle` 타입을 만들 수 있습니다. 그 결과, 대부분의 경우 `SafeHandle`이 아닌 타입은 파이널라이저를 구현할 필요가 없습니다. 대부분의 Dispose 패턴 구현은 결국 다른 관리 리소스를 래핑하게 되며, 이 중 일부는 `SafeHandle` 객체일 수 있습니다.

### 사용자 정의 SafeHandle을 사용한 Dispose 패턴 구현하기

다음 코드는 `SafeHandle`을 구현하여 관리되지 않는 리소스를 처리하는 방법을 보여줍니다.

```csharp
using System;
using System.IO;
using System.Runtime.InteropServices;
using Microsoft.Win32.SafeHandles;

// Wraps the IntPtr allocated by Marshal.AllocHGlobal() into a SafeHandle.
class LocalAllocHandle : SafeHandleZeroOrMinusOneIsInvalid
{
    private LocalAllocHandle() : base(ownsHandle: true) { }

    // No need to implement a finalizer - SafeHandle's finalizer will call ReleaseHandle for you.
    protected override bool ReleaseHandle()
    {
        Marshal.FreeHGlobal(handle);
        return true;
    }

    // Allocate bytes with Marshal.AllocHGlobal() and wrap the result into a SafeHandle.
    public static LocalAllocHandle Allocate(int numberOfBytes)
    {
        IntPtr nativeHandle = Marshal.AllocHGlobal(numberOfBytes);
        LocalAllocHandle safeHandle = new LocalAllocHandle();
        safeHandle.SetHandle(nativeHandle);
        return safeHandle;
    }
}

public class DisposableBaseWithSafeHandle : IDisposable
{
    // Detect redundant Dispose() calls.
    private bool _isDisposed;

    // Managed disposable objects owned by this class
    private LocalAllocHandle? _safeHandle = LocalAllocHandle.Allocate(10);
    private Stream? _otherUnmanagedResource = new MemoryStream();

    // Public implementation of Dispose pattern callable by consumers.
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }

    // Protected implementation of Dispose pattern.
    protected virtual void Dispose(bool disposing)
    {
        if (!_isDisposed)
        {
            _isDisposed = true;

            if (disposing)
            {
                // Dispose managed state.
                _otherUnmanagedResource?.Dispose();
                _safeHandle?.Dispose();
                _otherUnmanagedResource = null;
                _safeHandle = null;
            }
        }
    }
}
```

> 💡 참고
> 
> `DisposableBaseWithSafeHandle` 클래스의 동작은 이전 예제의 `DisposableBaseWithFinalizer` 클래스와 동일하지만, 여기에서 보여준 방식은 더 안전합니다:
> 
> - SafeHandle이 파이널라이제이션을 처리하므로 파이널라이저를 구현할 필요가 없습니다.
> 
> - 스레드 안전을 보장하기 위한 동기화가 필요하지 않습니다. `DisposableBaseWithSafeHandle`의 Dispose 구현에 경합 조건(race condition)이 있더라도, SafeHandle이 `SafeHandle.ReleaseHandle`이 정확히 한 번만 호출되도록 보장합니다.

## **.NET의 기본 제공 SafeHandle 목록**

다음은 [Microsoft.Win32.SafeHandles]([Microsoft.Win32.SafeHandles Namespace | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/microsoft.win32.safehandles?view=net-9.0)) 네임스페이스에서 제공하는 파생 클래스들로, 안전한 핸들을 제공합니다.

# 참조

- [MSDN: Cleaning up unmanaged resources]([Cleaning up unmanaged resources - .NET | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/unmanaged))

- [MSDN: Implement a Dispose method]([Implement a Dispose method - .NET | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/implementing-dispose))
