# UFUNCTION 노출 기준 정리

## 오늘의 핵심

`UFUNCTION`을 붙일지 말지는 "블루프린트에서 쓰는가?"만 보고 판단하면 안 된다.

더 중요한 기준은 이 함수가 값을 읽기만 하는지, 게임 상태를 바꾸는지, C++ 내부 구현용인지, BP에서 구현할 확장 지점인지다.

## 기본 판단 기준

```text
값만 읽는다              -> BlueprintPure
상태를 바꾸거나 실행한다 -> BlueprintCallable
C++ 내부에서만 쓴다      -> 일반 C++ 함수
BP가 구현할 연출 지점이다 -> BlueprintImplementableEvent
```

## BlueprintPure

`BlueprintPure`는 값을 읽거나 계산만 하고 게임 상태를 바꾸지 않는 함수에 사용한다.

```cpp
UFUNCTION(BlueprintPure)
float GetHealthRatio() const;

UFUNCTION(BlueprintPure)
bool CanDash() const;
```

예시:

```cpp
float GetHealthRatio() const
{
    return CurrentHealth / MaxHealth;
}
```

이 함수는 HP를 바꾸지 않고 비율만 계산한다.

UI에서 HP바에 사용하더라도, 상태를 변경하지 않고 읽기만 하므로 `BlueprintPure`가 더 적절하다.

## BlueprintCallable

`BlueprintCallable`은 호출했을 때 게임 상태가 바뀌거나, 명령처럼 실행되는 함수에 사용한다.

```cpp
UFUNCTION(BlueprintCallable)
void ApplyDamage(float Damage);

UFUNCTION(BlueprintCallable)
void StartDash();
```

예시:

```cpp
void ApplyDamage(float Damage)
{
    CurrentHealth -= Damage;
}
```

이 함수는 호출하면 HP가 줄어든다.

즉, 게임 상태를 변경하는 부작용이 있으므로 `BlueprintPure`는 부적절하다.

다만 상태 변경 함수를 무조건 블루프린트에 열면 위험할 수 있다.

예를 들어 UI 블루프린트에서 실수로 `ApplyDamage(10)`을 호출하면 실제 HP가 줄어든다.

그래서 블루프린트에서 호출할 필요가 확실할 때만 `BlueprintCallable`로 여는 것이 좋다.

## 일반 C++ 함수

블루프린트나 언리얼 리플렉션 시스템에서 알 필요가 없는 내부 구현용 함수는 일반 C++ 함수로 둔다.

```cpp
FVector CalculateDashDirection() const;
bool ConsumeStaminaInternal(float Amount);
void UpdateDashState(float DeltaTime);
```

예시:

```cpp
FVector CalculateDashDirection() const
{
    return GetActorForwardVector();
}
```

대시 방향 계산이 C++ 내부에서만 필요하다면 `UFUNCTION` 없이 일반 함수로 충분하다.

불필요하게 `UFUNCTION`을 붙이면 엔진 리플렉션 대상이 늘어나고, 함수의 의도도 흐려질 수 있다.

## BlueprintImplementableEvent

`BlueprintImplementableEvent`는 C++에서 흐름은 잡고, 실제 연출이나 확장 동작은 블루프린트에서 구현하게 하고 싶을 때 사용한다.

```cpp
UFUNCTION(BlueprintImplementableEvent)
void OnDashStarted();

UFUNCTION(BlueprintImplementableEvent)
void OnHitEffectRequested();
```

예시:

```cpp
void ApplyDamage(float Damage)
{
    CurrentHealth -= Damage;
    OnHitEffectRequested();
}
```

C++은 데미지 처리만 담당하고, BP는 피격 이펙트, 사운드, 카메라 흔들림 같은 연출을 붙일 수 있다.

## 오늘 교정한 생각

처음에는 이렇게 생각했다.

```text
블루프린트에서 쓰면 BlueprintCallable
```

하지만 더 정확한 기준은 이렇다.

```text
블루프린트에서 읽기만 한다 -> BlueprintPure
블루프린트에서 실행한다   -> BlueprintCallable
```

즉, `GetHealthRatio()`는 UI에서 쓰이더라도 값을 읽기만 하므로 `BlueprintPure`가 더 적절하다.

반대로 `ApplyDamage()`는 호출하면 HP가 바뀌므로 `BlueprintCallable` 후보가 된다.

## 한 줄 정리

`UFUNCTION`은 붙일 수 있는 곳에 전부 붙이는 매크로가 아니라, C++ 함수를 언리얼 엔진 시스템에 어떤 방식으로 노출할지 결정하는 계약이다.

