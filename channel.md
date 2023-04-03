# 채널

서로 다른 고루틴이 통신하는 통로

생성 : `make(chan [value-type])`

값 수신/전달 : `channel ←`, `←channel`

채널 종료: `close(channel)`

- Unbuffered Channel
    
    ex. `make(chan int)`
    
    - 전송보장 필요 시
    - 수신부는 수신할 데이터가 있을 때까지 블락
        
        송신부는 수신부가 값을 받을 때까지 블락
        
    
- Buffered Channel
    
    ex. `make(chan int, 100)`
    
    - 전송보장 필요 x 시에 - send와 receive가 정확하게 동기화 x
    - 수신부는 수신할 데이터가 있을 때까지 블락
        
        송신부는 값이 버퍼에 복사될때까지 블락
        
    

## 상태

상태는 nil, open, close 3가지 상태로 나뉨

### **nil**

`var ch chan struct{}`

모든 send, receive 요청이 blocking되며 close 요청 시엔 panic 발생

**사용 사례**

```go
func useNilInSelect(c chan string) {
    for {
        select {
        case n := <-c:
            fmt.Println(n)
            c = nil
        case <-time.After(time.Second * 1):
            fmt.Println("Timeout - select return")
            return
        }
    }
}
```

c = nil 이 없다면 1초가 지날 때까지 채널에 들어오는 값을 모두 출력하지만 c = nil로 인해 한 번만 출력되고 프로그램이 종료

### open

`make(chan struct{})`

모든 send, receive 요청이 허용

### close

`close(ch)`

- 채널 버퍼의 값을 수신하는 것은 가능하지만 값 send 요청 시 panic 발생
- 버퍼의 값을 모두 돌려준 이후에는 제로값을 돌려줌

```go
func main() {
	ch := make(chan int)

	go func() {
		ch <- 1
	}()

	v, ok := <-ch
	fmt.Printf("%d, %t\n", v, ok)//1, true

	close(ch)
	v, ok = <-ch
	fmt.Printf("%d, %t\n", v, ok)//0, false
}
```

- closed 된 채널을 close 시 panic 발생
    - 따라서 close가 여러번 발생할 가능성이 있는 코드에서는 sync.Once 등을 사용해 한번만 호출되도록 구현

```go
func main() {
	ch := make(chan int)
	close(ch)
	close(ch)
} //panic: close of closed channel

func main(){
	once := sync.Once{}
	ch := make(chan int)
	for i := 0; i < 5; i++ {
		once.Do(func() {
			close(ch)
			fmt.Printf("try to close channel %d times\n", i+1)
		})
	}
}
//try to close channel 1 times\n
```

### 고루틴 누수 방지

- 채널이 고루틴 내에서 데이터를 받거나 주기 위해 대기하는 경우
    - 이후에 고루틴을 사용하지 않더라도 고루틴은 계속 자원을 필요로 하며 런타임에 의해 가비지 컬렉션되지 않음 ⇒ 부모 고루틴이 자식 고루틴에게 보내는 취소 신호 정의

### 기본 사용법

**notification / timeout**

```go
//timeout

go func() {
    done <- struct{}{}
}()

select {
case <-done:
	return true
case <-time.After(10 * time.Second)
  return false
}
```

- 작업 완료(done)나 타임아웃(time.After) 등의 알림 기능 구현

**for-select**

```jsx
//채널에 반복적으로 송신
for _, s := range[] string {"a", "b", "c"} {
	select {
		case <-done:
			return
		case stringStream <- s:
	}
}

//멈출때까지 무한히 대기/작업
for {
	select {
		case <- done:
			return
		default:
		// 작업
	}
	// 작업
}
```

**Mutex Lock (unbuffered), Semaphore (buffered)**

```go
increase := func() {
		mutex <- struct{}{} // lock
		counter++
		<-mutex // unlock
}
```

- 채널로 공유 메모리 자원에 락을 걸어 Mutex, Semaphore 구현

## Chan Chan

<aside>
💡 채널 역시 하나의 자료형으로써 채널 타입을 받는 채널이 문법적으로 가능하다.

</aside>

- chan chan을 통해 양방향 처리가 가능

**Pub-Sub패턴**

- publisher 는 subscriber에게 직접 메시지를 보내도록 프로그래밍하는 것이 아닌 subscriber에 대한 지식 없이 클래스로 분류
- subscriber는 하나 이상의 클래스에 대해 관심을 표현하고 관심 있는 메시지만 수신
- 장점 :  관심사의 분리, 확장성
- 단점 : subscriber가 메시지를 제대로 받았는지 publisher가 알 수 없음 → 추가적인 로깅 서버 필요

blog.go

```go
package main

import "fmt"

type Post struct {
	Title string
	Content string
}

type blog struct {
	name string
	subscribers map[chan<- string] struct{}
	subscribe chan chan<- string
	unsubscribe chan chan<- string
	publish chan Post
}

func Blog(name string) *blog {
	b := new(blog)
	b.subscribers = make(map[chan<- string] struct{}) //구독자 채널을 저장
	b.subscribe = make(chan chan<- string) //구독하는 유저의 채널을 받는 채널
	b.unsubscribe = make(chan chan<- string)  //구독 취소하는 유저의 채널을 받는 채널
	b.publish = make(chan Post) //구독한 유저에게 Post를 보내는 채널
	b.name = name
	go b.run()
	return b
}

func (b *blog) SubscribedBy(subscriber chan<- string) {
	b.subscribe <- subscriber 
} //유저의 채널을 받아 해당하는 채널에 전달

func (b *blog) UnsubscribedBy(subscriber chan<- string) {
	b.unsubscribe <- subscriber
} //유저의 채널을 받아 해당하는 채널에 전달

func (b *blog) Publish(p Post) {
	b.publish <- p
}

func (b *blog) run() {
	for {
		select {
		case sub := <-b.subscribe:
			b.subscribers[sub] = struct{}{}
		case sub := <-b.unsubscribe:
			delete(b.subscribers, sub)
		case p := <-b.publish:
			fmt.Printf("%s published %s\n", b.name, p.Title)
			for subscriber := range b.subscribers {
				subscriber <- p.Title //구독한 유저들의 채널에 데이터 전달
			}
		}
	}
}
```

user.go

```go
package main

import "fmt"

type user struct {
	id string
	notifier chan string
}

func User(id string) *user {
	u := new(user)
	u.id = id
	u.notifier = make(chan string) //blog에 전달할 유저 채널
	go u.listen()
	return u
}

//blog 의 메서드를 통해 유저 채널 전달
func (u *user) Subscribe(b *blog) {
	b.SubscribedBy(u.notifier)
}

func (u *user) Unsubscribe(b *blog) {
	b.UnsubscribedBy(u.notifier)
}

func (u *user) listen() {
	for {
		s := <- u.notifier //blog에 전달한 유저의 채널에 값이 전달될 경우 작업
		fmt.Printf("%s received new post - %s\n", u.id, s)
	}
}
```

**순서가 보장된 Worker Pool**

- 할당된 작업을 처리하려는 고루틴의 집합
- 작업의 수만큼 worker를 생성하여 처리 ⇒ 과부하 발생 가능성
- 할당할 워커의 수를 조정해 작업하도록 구현

```go
package fetcher

import (
	"io"
)

type MultiFetcher struct {
	*fetcher
	workers int
}

func Multi(writer io.Writer, amount int) *MultiFetcher {
	return &MultiFetcher{fetcher: Simple(writer), workers: amount}
}

func (m *MultiFetcher) Run() {

	readCh := make(chan string)
	fetchCh := make(chan chan string, m.workers)
	distributeCh := make(chan string)

	
	go m.read(readCh)
	go m.fetch(readCh, fetchCh)
	go m.distribute(fetchCh, distributeCh)
	m.print(distributeCh)

}

func (M *MultiFetcher) read(outCh chan<- string){
	defer close(outCh)
	for i := 0; i < 1000; i ++ {
		time.Sleep(time.Duration(rand.Intn(5)) * time.Millisecond)
		outCh <- fmt.Sprintf("line: %d", i+1)
	}
}

func (m *MultiFetcher) fetch(inCh <-chan string, outChCh chan<- chan string) {

	defer close(outChCh)

	for line := range inCh {

		outCh := make(chan string)

		go m.fetchLine(line, outCh)
		outChCh <- outCh
	}
}

func (m *MultiFetcher) distribute(inCh <-chan chan string, outCh chan<- string) {
	defer close(outCh)
	for ch := range inCh {
		outCh <- <- ch
	}
}

func (m *MultiFetcher) fetchLine(line string, outCh chan<- string) {
	defer close(outCh)
	m.fetcher.fetchLine(line, outCh) // outCh <- line
}
```

기존의 워커풀이 idle한 워커가 생기면 작업을 배정해 효율적으로 처리하는 대신 작업의 순서가 뒤죽박죽이었다면,

chan chan을 사용한 이 워커풀은 fetch 함수의

```go
for line := range inCh {

		outCh := make(chan string)

		go m.fetchLine(line, outCh)
		outChCh <- outCh
	}
```

위 코드에 의해 앞 작업이 끝나지 않으면 뒷작업이 일찍 끝나더라도 통과되지 않음으로써 순서대로 작업이 진행된다.

[https://simplear.tistory.com/31?category=883241](https://simplear.tistory.com/31?category=883241)

**처리 대기**

```go
package daemon

import (
	"fmt"
	"math/rand"
	"time"
)

type daemon struct {
	req chan chan struct{}
}

func New() *daemon {
	d := new(daemon)
	d.req = make(chan chan struct{})
	return d
}

func (d *daemon) StartLoop() {
	go func() {
		for {
			select {
			case ch := <-d.req:
				rand.Seed(time.Now().UnixNano())
				time.Sleep(500 * time.Millisecond)
				fmt.Println("work")//작업
				close(ch)
			}
		}
	}()
}

func (d *daemon) Do() {
	ch := make(chan struct{})
	d.req <- ch //요청 보내기
	<-ch //처리 종료 대기
}

```

```go
func main() {

    d := daemon.New()

    d.StartLoop()

    d.Do()

    fmt.Println("done")
}
```

- `StartLoop()` : channel을 받아 작업 실행 후 channel 종료
- `Do()` : 작업을 실행 후 channel이 종료될 때까지 대기
- request / response 와 비슷한 역할