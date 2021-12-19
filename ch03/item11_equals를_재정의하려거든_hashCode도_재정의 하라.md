## 아이템11. equals를 재정의하려거든 hashCode도 재정의 하라
▶ equals를 제정의한 클래스 모두에서 hashCode도 재정의해야 한다. 그렇지 않으면 hashCode 일반 규약을 어기게 되어 해당 클래스의 인스턴스를 HashMap이나 HashSet 같은 컬렉션의 원소로 사용할 때 문제가 발생한다.

### hashCode 일반 규약
**1. ```equals``` 비교에 사용되는 정보가 변경되지 않았다면 ```hashCode```메서드는 항상 일관된 값을 반환해야 한다.**
단, 애플리케이션을 다시 실행한다면 이 값이 달라져도 상관없음

**2. ```equals```가 두 객체를 같다고 판단했다면 두 객체의 ```hashCode```는 똑같은 값을 반환해야 한다.**
▷ 논리적으로 같은 객체는 같은 해시코드를 반환해야 됨

**3. ```equals```가 두 객체를 다르다고 판단했더라도, 두 객체의 ```hashCode```가 서로 다른 값을 반환할 필요는 없다.**
단, 다른 객체에 대해서는 다른값을 반환해야 해시 테이블의 성능이 좋아짐

### 논리적으로 같은 객체는 같은 해시코드를 반환해야 한다.
```equals```는 물리적으로 다른 두 객체를 논리적으로는 같다고 할 수 있다. 그러나 hashCode 메서드는 이 둘이 전혀 다르다고 판단하여, 규약과 달리 서로 다른 값을 반환한다.
```Java
Map<PhoneNumber, String> m = new HashMap<>();
m.put(new PhoneNumber(100, 200, 300), "하하");
m.get(new PhoneNumber(100, 200, 300)); // ▶ 결과 : 하하(x) null(o)
```
```PhoneNumber```클래스는 ```hashCode```를 재정의하지 않았기 때문에, 논리적 동치인 두 객체가 서로 다른 해시코드를 반환하여 ```get``` 메서드는 엉뚱한 해시 버킷에 가서 객체를 찾으려 한 것이다.

### ❌적법하지만 최악의 hashCode 구현
```Java
@Override
public int hashCode() {
	return 42;
}
```
동치인 모든 객체에서 똑같은 해시코드를 반환해 적법해 보이지만, 모든 객체에게 똑같은 값만 내어주므로 모든 객체가 해시테이블의 버킷 하나에 담겨 마치 연결리스트처럼 동작 → 느려짐 + 객체 많아짐 ▶ 최악

### 좋은 해시 함수: 서로 다른 인스턴스에 다른 해시코드를 반환
이상적인 해시 함수는 주어진 (서로 다른) 인스턴스들을 32비트 정수 범위에 균일하게 분배해야 한다.
```Java
@Override
public int hashCode() {
    int c = 31;
    //1. int변수 result를 선언한 후 첫번째 핵심 필드에 대한 hashcode로 초기화 한다.
    int result = Integer.hashCode(firstNumber);

    //2. 기본타입 필드라면 Type.hashCode()를 실행한다. Type은 기본타입의 박싱 클래스이다.
    result = c * result + Integer.hashCode(secondNumber);

    //3. 참조타입이라면 참조타입에 대한 hashcode 함수를 호출 한다.
    //4. 값이 null이면 0을 더해 준다.
    result = c * result + address == null ? 0 : address.hashCode();

    //5. 필드가 배열이라면 핵심 원소를 각각 하나의 원소처럼 갱신한다.
    for (String elem : arr) {
      result = c * result + elem == null ? 0 : elem.hashCode();
    }

    //6. 배열의 모든 원소가 핵심필드이면 Arrays.hashCode를 이용한다.
    result = c * result + Arrays.hashCode(arr);

    //7. result = 31 * result + c 형태로 초기화 하여 result를 반환한다.
    return result;
}
```
#### 주의할 점
- ```equals```비교에 사용되는 필드에 대해서만 해시코드를 계산한다.
- 성능을 높이기위해 해시코드를 계산할 때 핵심 필드를 생략해서는 안된다.
- hash로 바꾸려는 필드가 기본 타임이 아니면 해당 필드의 hashCode를 불러 구현한다.
계산이 복잡한 경우에는 표준형을 만들어 구현
- 참조 타입 필드가 null인 경우 0을 사용
- 곱하는 수가 31로 정한 이유는 31이 홀수이면서 소수(prime)이기 때문 

**전형적인 hashCode 메서드**
```Java
@Override
public int hashCode() {
	int result = short.hashCode(areaCode);

    result = 31 * result + Short.hashCode(prefix);
    result = 31 * result + Short.hashCode(lineNum);

    return result;
}
```
```PhoneNumber``` 인스턴스의 핵심 필드 3개를 사용해 간단한 계산을 수행하고, 이 과정에서 비결정적 요소는 없다.
▷ 동치인 ```PhoneNumber``` 인스턴스는 서로 같은 해시코드르 가질 것이 확실하다.

**한 줄자리 hashCode 메서드 - 성능이 살짝 아쉽다.**
```Java
@Override
public int hashCode() {
	return Objects.hash(lineNum, prefix, areaCode);
}
```
입력 인수를 담기 위한 배열이 만들어지고, 입력 중 기본 타입이 있다면 박싱과 언박싱도 거친다. 속도가 느리다.

### hashCode의 캐싱과 지연 초기화
- 클래스가 불변이고, 해시코드를 계산하는 비용이 크다면 매번 새로 계산하기 보다 캐싱을 고려해야 한다.
▷ 이 타입의 객체가 주로 해시의 키로 사용될 것 같다면 인스턴스가 만들어질 때 해시코드를 계산해둬야 한다.
- 해시의 키로 사용되지 않는 경우라면 hashCode가 처음 불릴 때 계산하는 지연 초기화를 하면 좋다.
▷ 필드를 지연 초기화하려면 그 클래스를 스레드 안전하게 만들도록 신경 써야 한다.

**해시코드를 지연 초기화하는 hashCode 메서드 - 스레드 안정성까지 고려해야 한다.**
```Java
private int hashCode;

@Override
public int hashCode() {
	int result = hashCode;
	if (result == 0) {
      result = Short.hashCode(areaCode);
      result = 31 * result + Short.hashCode(prefix);
      result = 31 * result + Short.hashCode(lineNum);
      hashCode = result;
    }
    return result;
}
```

### 핵심정리
```equals```를 재정의할 때는 ```hashCode```도 반드시 재정의해야 한다. 그렇지 않으면 프로그램이 제대로 동작하지 않을 것이다. 재정의한 ```hashCode```는 ```Object```의 API문서에 기술된 일반 규약을 따라야 하며, 서로 다른 인스턴스라면 되도록 해시코드도 서로 다르게 구현해야 한다.