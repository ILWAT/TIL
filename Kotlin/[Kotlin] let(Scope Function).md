# [Kotlin] let(Scope Function)

### Scope Function

객체 Context 내에서 코드 블록을 실행하는 것을 유일한 목표로 삼는 함수

### let

스코프 내부에서 it을 통해 해당 객체를 사용할 수 있다.

return 값은 람다의 결과이다. 아래 예

1. null을 가질 수 있는 객체를 안정적으로 사용하기 위해 사용
    
    ```kotlin
    val str: String? = "Hello"   
    //processNonNullString(str)       // compilation error: str can be null
    val length = str?.let { 
        println("let() called on $it")        
        processNonNullString(it)      // OK: 'it' is not null inside '?.let { }'
        it.length
    }
    ```
    
    ?.(safe call)의 형태로 사용하면 null을 가질 수 있는 객체가 null이 아닌 경우에만  let에 넘겨진 scope가 실행한다.
    
2. let은 콜 체인 상황에서 이상의 함수를 호출하기 위해 사용된다.
    
    ```kotlin
    val numbers = mutableListOf("one", "two", "three", "four", "five")
    numbers.map { it.length }.filter { it > 3 }.let { 
        println(it)
        // and more function calls if needed
    } 
    ```
    

### References

[Scope functions | Kotlin](https://kotlinlang.org/docs/scope-functions.html)

[**[Kotlin] let, also, run, with, apply.. 뭐가 다른데?**](https://velog.io/@jisoo0817/let-also-run-with-apply..-%EB%AD%90%EA%B0%80-%EB%8B%A4%EB%A5%B8%EB%8D%B0)