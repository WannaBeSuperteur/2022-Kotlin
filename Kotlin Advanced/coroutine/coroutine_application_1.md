## 요구 사항 분석
사용자가 앱의 특정 화면을 열면 사용자의 습관 달성 통계를 앱의 상단 박스에 출력한다. 이 통계 자료는 서버의 응답 데이터를 기초로 만들어지는 것이지만, 이 화면을 여는 것과 함께 실행되는 함수에서 실행하기 때문에 서버 응답 수신 이후로 지연시킬 수 없다.

우리는 서버 응답이 도착한 후에, 그것을 바탕으로 앱의 상단 박스에 습관 달성 통계를 출력해야 한다. 그러나 앱의 상단 박스에 습관 달성 통계를 표시하는 함수가 서버 응답 이전에 먼저 실행될 수 있다.

그 함수가 항상 서버 응답 이후에 실행되도록 ```coroutine```을 이용하여 코드를 재설계하라.

### 기본 코드 구조
```kotlin
import kotlinx.coroutines.*
import java.util.Random

var routineData = listOf<Int>() // 날짜별 달성한 습관의 개수를 날짜 순서대로 저장

fun main() {
    afterOpenScreen()
}

fun afterOpenScreen() = runBlocking {
    val random = Random()
    val t0 = 200 + random.nextInt(300) // 서버 응답 시간
    val t1 = 250 + random.nextInt(200) // 화면 로딩 시간
    
    launch {
        delay(t0.toLong())
        handleServerAnswer(t0) // 서버 응답 처리 함수
    }
    launch {
        delay(t1.toLong())
        handleLoadingRoutineStat(t1) // 습관 통계 로딩 함수
    }
}

fun handleServerAnswer(time: Int) {
    println("handling server answer in ${time} msec ...")
    
    // routineData를 갱신하는 코드 작성
    // 임의의 데이터로 갱신
}

fun handleLoadingRoutineStat(time: Int) {
    println("handling loading routine statistics in ${time} msec ...")
    
    // routineData를 이용하여 통계를 출력하는 코드 작성
    // routineData = [1, 4, 4, 2, 5, 4, 3, 4] 일 때,
    
    // 평균 3.375, 최소 1, 최대 5
    // 습관    일수
    // 1       1
    // 2       1
    // 3       1
    // 4       4
    // 5       1
    
    // 와 같이 출력할 것.
}
```

## 예제 코드
```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*
import java.util.Random

// var routineData = listOf<Int>() // 날짜별 달성한 습관의 개수를 날짜 순서대로 저장
val channel = Channel<List<Int>>() // 데이터 송수신용 채널

fun main() {
    afterOpenScreen()
}

fun afterOpenScreen() = runBlocking {
    val random = Random()
    val t0 = 200 + random.nextInt(300) // 서버 응답 시간
    val t1 = 250 + random.nextInt(200) // 화면 로딩 시간
    
    launch {
        delay(t0.toLong())
        handleServerAnswer(t0) // 서버 응답 처리 함수
    }
    launch {
        delay(t1.toLong())
        handleLoadingRoutineStat(t1) // 습관 통계 로딩 함수
    }
}

fun handleServerAnswer(time: Int) = runBlocking {
    println("handling server answer in ${time} msecs ...")
    
    // routineData를 갱신 (또는 초기화)
    val routineData = listOf(2, 5, 4, 4, 5, 2, 4, 3, 1, 4, 4, 5, 2, 5)
    
    // 채널로 데이터 전송
    launch {
        channel.send(routineData)
    }
}

fun handleLoadingRoutineStat(time: Int) = runBlocking {
    println("handling loading routine statistics in ${time} msecs ...")
    
    launch {
        val start = System.currentTimeMillis()
        val updatedRoutineData = channel.receive() as List<Int> // 채널의 데이터 수신
        val end = System.currentTimeMillis()
        
        println("time used for channel receiving : ${end - start} msecs")
        
        with (updatedRoutineData) {
            println("\n평균 ${sum().toDouble() / size.toDouble()}, 최소 ${min()}, 최대 ${max()}")
            println("습관    일수")
            for (i in min() .. max()) {
                println("${i}       ${count {it == i}}")
            }
        }
    }
}
```

## 예제 출력
```kotlin
handling server answer in 388 msecs ...
handling loading routine statistics in 428 msecs ...
time used for channel receiving : 1 msecs

평균 3.5714285714285716, 최소 1, 최대 5
습관    일수
1       1
2       3
3       1
4       5
5       4
```

```kotlin
handling loading routine statistics in 384 msecs ...
handling server answer in 425 msecs ...
time used for channel receiving : 46 msecs

평균 3.5714285714285716, 최소 1, 최대 5
습관    일수
1       1
2       3
3       1
4       5
5       4
```
