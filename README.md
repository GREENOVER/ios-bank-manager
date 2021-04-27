# iOS Bank Manager Application Project
### 은행 창구 업무 및 전체 은행에 대한 관리 기능을 구현한 프로젝트
[Ground Rule](https://github.com/GREENOVER/ios-bank-manager/blob/main/GroundRule.md)
***
#### What I learned✍️
- GCD
- DispatchQueue
- Operation
- willSet
- DispatchSemaphore
- Sync & Asynce
- CFAbsoluteTimeGetCurrent
- LocalizedDescription
- Higher-order function
- LocalizedError

#### What have I done🧑🏻‍💻
- GCD와 오퍼레이션의 특징과 차이를 학습하였다.
- 디스패치큐를 통해 은행 업무가 비동기적으로 처리되도록 구현하였다.
- 세마포어를 학습하고 사용하여 전체 디스패치큐 그룹의 끝나는 시간을 계산하도록 구현하였다.
- 동기와 비동기의 차이를 학습하고 기능에 따라 구현하였다.
- 중첩된 열거타입을 정의하였다.
- 시스템의 현재시간을 구하고 전체 업무시간을 계산하도록 기능을 구현하였다.
- 에러의 지역화된 설명을 사용하였다.
- 고차함수의 filter, sort, map 등을 학습하고 구현하였다.
- 각 상황에 맞는 에러 처리를 구현하였다.


#### Trouble Shooting👨‍🔧
- 문제점 (1)
  - 은행원이 일을 처리할때 고객의 업무 시작 알림보다 종료 알림이 더 빠르게 뜨는 문제 발생
- 원인
  - 일을 시작하는 work와 일이 끝난걸 알리는 doneworking 메서드가 비동기적으로 스레드를 잡고 동작하기에 발생하는 문제이다.
- 해결방안
  - 이 부분을 세마포어를 사용하여 해당 고객의 업무 종료부분에 wait를 해주고 업무가 시작되고 서류 검토가 완료된 후 signal을 보내도록 수정하였다.
  ```swift
  func work() {
      guard let customer = customer else {
          print("고객이 없습니다!")
          return
      }
      print("\(customer.waiting)번 \(customer.priority.describing)고객 \(customer.businessType.rawValue) 업무 시작")
      
      if customer.businessType == .loan {
          print("\(customer.waiting)번 \(customer.priority.describing)고객 대출서류 검토 시작")
          waitExamineLoan(customer)
          print("\(customer.waiting)번 \(customer.priority.describing)고객 대출서류 검토 완료")
      }
      semaphore.signal()
  }
  ```
  ```swift
  private func doneWorking() {
      if let customer = customer {
          semaphore.wait()
          print("\(customer.waiting)번 \(customer.priority.describing)고객 \(customer.businessType.rawValue) 업무 종료")
      }
  }  
  ```
- 문제점 (2)
  - 은행원의 대출 심사를 위해 DispatchQueue 사용을 하고싶은데 컴파일 에러가 나는 문제
- 원인
  - 처음 Banker(은행원)을 구조체 타입으로 정의하였는데, 구조체 타입은 DispatchQueue를 사용하는 과정에서 task 클로저가 @escaping() -> void 형식이므로, mutating 메서드를 사용하면 @escaping 클로저를 사용할 수 없다. 즉 은행원이 구조체 타입이라 탈출 클로저의 사용에 제약이 생겼다.
- 해결방안
  - 구조체 타입의 은행원을 클래스 타입으로 수정하여 DispatchQueue를 통한 비동기 업무처리를 구현하였다. (네트워크 통신과 비동기 작업은 탈출 클로저를 핸들링 하는것이 많단걸 배웠다.)
  ```swift
  class Banker {
      private func waitExamineLoan(_ customer: Customer) {
        Thread.sleep(forTimeInterval: 0.3)
        DispatchQueue.global().sync {
            HeadOffice.main.examineLoan(customer)
        }
        Thread.sleep(forTimeInterval: 0.3)
    }
  }
  ```
- 문제점 (3)
  - 각 은행원이 본사로 대출 서류 심사를 보내면 순서대로 처리가 되지 않는 문제 발생
- 원인
  - 본사에서 대출 서류 심사에 대해 Thread.sleep으로 시간 지연은 구현되었는데 밀려온 각 스레드에 대해 동기적으로 처리되지 않고 비동기적으로 처리가 되어 발생한 문제이다.
- 해결방안
  - 본사로 넘어온 스레드들을 디스패치큐 레이블로 한 그룹으로 묶어주고 해당 그룹에 있는 업무들을 동기적으로 처리되도록 수정하였다.
  ```swift
  class HeadOffice {
    static let main = HeadOffice()
    
    let examineLoanQueue = DispatchQueue(label: "examineLoanQueue")
    
    private init() {}
    
    func examineLoan(_ customer: Customer) {
        self.examineLoanQueue.sync {
            print("\(customer.waiting)번 본사 대출서류 심사")
            Thread.sleep(forTimeInterval: 0.5)
          }
      }
  }
  ```

#### Thinking Point🤔
- 고민점 (1)
  - "fatalError는 앱이 크래시 날 경우를 내포하고 있으므로 되도록 사용하지 않는 것이 좋습니다. 다른 방식으로 에러 핸들링은 없을까요?"
  ```swift
  guard let userInput = Int(readLine()!) else {
      fatalError("잘못된 입력값 입니다!")
  }
  ```
- 원인 및 대책
  - 에러에 대한 처리 케이스를 따로 분리하여 LocalizedError를 상속하고 각 상황에 맞는 errorDescription을 핸들링 해줄 수 있도록 변경하였다.
  ```swift
  enum InputError: Error {
    case wrongInput
    case noInput
    case unknown
  }
  extension InputError: LocalizedError {
    var errorDescription: String? {
        switch self {
        case .wrongInput:
            return "잘못된 입력입니다. 다시 입력하세요!"
        case .noInput:
            return "입력값이 없습니다. 다시 입력하세요!"
        case .unknown:
            return "알 수 없는 입력값입니다. 다시 입력하세요!"
         }
      }
  }
  ```
  그 후 에러 처리를 해주는 부분에서 아래와 같이 코드를 수정하여 적절한 에러 처리가 되도록 해주었다.
  ```swift
  guard let userInput = readLine() else {
      print(InputError.wrongInput.localizedDescription)
      return startBank()
  }
  ```
- 고민점 (2)
  - "은행 업무 주체에 대해 고민해볼까요?"
  ```swift
  mutating func configureCustomers(numberOfCustomers: UInt) {
      for number in 1...numberOfCustomers {
          customers.append(Customer(waiting: number, taskTime: taskTime))
      }
  }
  ```
- 원인 및 대책
  - 위 코드를 처음에 은행매니저에서 구현해줘 고객의 수에 대해 은행의 역할로 주어졌는데 생각해보니 은행은 몇명의 고객이 올지 모르기때문에 해당 부분을 main에서 구현하도록 수정하였다.
  또한 업무 시작과 종료를 은행의 역할로 주었는데 은행원이 남은 고객이 있는지 판별하고 업무를 종료하도록 은행받아 확인하도록 구현하였다.
  ```swift
  func openBank() {
      calculateTotalTime {
          while !customers.isEmpty {
              for windowNumber in 0..<bankers.count {
                  checkBankerIsWorking(windowNumber)
              }
          }
          checkEnd()
          semaphore.wait()
      }
      initializeInfo()
  }
  ```
- 고민점 (3)
  - "customer가 꼭 필요한 메서드라면 guard를 사용하는 것이 의미전달이나 가독성 면에서 더 낫지 않을까?"
  ```swift
  func work() {
      if let customer = customer {
      }
  }
  ```
- 원인 및 대책
  - 고객은 꼭 필요하기에 if let은 조건으로 바인딩 해줄때 사용하는데 적합하지 않아 만약 고객이 없는데 호출이 되는 상황에서는 에러를 나타낼 수 있도록 guard let으로 변경하였다
  ```swift
  func work() {
      guard let customer = customer else {
          print("고객이 없습니다!")
          return
      }
  }
  ```
- 고민점 (4)
  - "CustomerPriority를 Customer 타입을 더 명확하고 타입 이름도 짧게 구현할 수 있는 방법이 있을까?"
  ```swift
  struct Customer {
    var waiting: UInt
    var taskTime: Double
    var priority: CustomerPriority
  }
  ```
- 원인 및 대책
  - Nested Type으로 구현하는것이 더 명확해보인다. 또한 외부에서도 고객의 우선순위라는것을 명확히 볼 수 있게 변경해보았다.
  ```swift
  struct Customer {
    enum Priority: Int, CaseIterable {
        case vvip = 0
        case vip = 1
        case normal = 2
        
        var describing: String {
            switch self {
            case .vvip:
                return "VVIP"
            case .vip:
                return "VIP"
            case .normal:
                return "일반"
            }
        }
    }
    var waiting: UInt
    var taskTime: Double
    var businessType: BusinessType
    var priority: Customer.Priority
  }
  ```
- 고민점 (5)
  - "세마포어의 value가 0인 것은 어떤것을 의미할까?"
  ```swift
  private var semaphore = DispatchSemaphore(value: 0)
  ```
- 원인 및 대책
  - value의 값은 여러 용도로 사용될 수 있는데 각 스레드를 동기 시키기 위해 0을 넣어주었다. wait가 들어오면 1씩 감소시키고 signal이 동작하면 증가시켜 깨어나는 시점을 동기화 할 수 있다. (wait를 먼저 만나게 구조적으로 구현하여 value의 값을 0을 주어 음수가 되면 스레드의 기능이 멈추고 대기하게하는것이 맞다.)
- 고민점 (6)
  - "아래 코드를 더 안전하고 좋은 코드로 바꿔볼 수 없을까?"
  ```swift
  let businessType:Int = Int.random(in: 1...2)
  taskTime = timeSetting(businessType)
  customers.append(Customer(waiting: number, taskTime: taskTime, priority: CustomerPriority(rawValue: Int.random(in: 1...3))!, businessType: BusinessType(rawValue: businessType)!))
  ```
- 원인 및 대책
  - 아래와 같이 구조적으로 나눠 좀 더 안전한 코드로 발전시켰다.
  ```swift
  let taskTime = duration(businessType)
  let customer = Customer(waiting: number, taskTime: taskTime, businessType: businessType, priority: customerPriority)
  customers.append(customer)
  ```
- 고민점 (7)
  - "CFAbsoluteTimeGetCurrent 함수를 사용하는것이 Date, TimeInterval 등의 타입을 사용했을 때 보다 어떤 이점이 있을까?
  ```swift
  private func calculateTotalTime(bankTaskFunction: () -> ()) {
    let startTime = CFAbsoluteTimeGetCurrent()
  }
- 원인 및 대책
  - 시스템의 절대 시간을 이용하는 함수로 보다 정확한 시간 계산을 할 수 있다. Date 타입을 가지고 위에서 작성한 코드처럼 startTime을 TimeInterval로 설정하고 endTime을 구해줌으로 시간을 계산해줄 수도 있지만 해당 타입은 날짜를 계산해줄때 더 많이 사용된다. 즉 프로세스가 돌아갈때 절대시간을 Date 타입을 활용한것보다 더 간편하고 정확하게 구현할 수 있다.



#### InApp 📱
![시연영상](https://user-images.githubusercontent.com/72292617/116229222-323eb900-a791-11eb-80e7-418a0fe38516.gif)


