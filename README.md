# 날씨 앱
이름, 나이, 전화번호를 가지는 연락처를 추가, 삭제, 수정할 수 있는 연락처 앱
<br/>

# 구현 영상
![화면 기록 2023-12-22 오전 10 16 40](https://github.com/tasty-code/ios-weather-forecast/assets/52391722/6490db7f-c3e4-4414-aef2-1f6fb92e1c67)




## 파일구조📁
- Controller
    - WeatherViewController
- Data
    - WeatherDataManagerDelegate
    - WeatherDataManager
    - Location
        - LocationDataManagerDelegate
        - LocationDataManager
- Extensoins
    - Double
- Error
    - NetworkError
- Location
    - LocationDataManagerDelegate
    - LocationDataManager
- Model
    - WeatherToday
    - WeatherForecast
    - WeatherCommonInformation
- Network
    - WeatherNetworkManagerDelegate
    - WeatherNetworkManager
    - ImageCacheManager
- Service
    - ServiceType
    - DataServiceable
    - ForecastDataServiceDelegate
    - ForecastDataService
    - TodayDataServiceDelegate
    - TodayDataService
    - IconDataServiceDelegate
    - IconDataService
- View
    - WeatherCollectionViewCell
    - WeatherCollectionHeaderView
- Util
    - Contants


<br/>

## 주요 클래스 설명

| 이름 | 타입 | 구현내용 |
| --- | --- | --- |
| WeatherViewController | Class | View의 Binding과 네트워크를 통해 데이터를 받아왔을 때의 처리 |
| WeatherDataManager | Class | 날씨 관련 데이터들을 관리하는 클래스 |
| DataServiceable | Protocol | 식별 id, 은행 업무 타입을 프로퍼티로 가지고 있는 고객 |
| ForecastDataService | Class | 5일 날씨 예보 데이터를 decode하는 역할 |
| TodayDataService | Class | 현재 날씨 예보 데이터를 decode하는 역할 |
| IconDataService | Class | data를 UIImage로 변환 및 Cache에 저장하는 역할  |
| LocationDataManager | Class | 좌표값, 주소 정보를 CoreLocation을 사용해서 받아오는 역할 |


<br/>

# 구현 내용
### STEP1
- API 서버와 통신하기 위한 데이터 모델 구현
- 범용성, 재사용성을 고려한 네트워킹 타입 구현 (with URLSession)

### STEP2
- CoreLocation을 통해 현재 좌표값, 주소정보 받아오는 기능 구현
- weather api를 통해 현재 날씨와 5일 예보 날씨 데이터 받아오는 기능 구현
- STEP1에서 구현해 놓은 데이터 모델에 매칭

### STEP3
- CollectoinView을 활용하여 UI 구현 (using CollectionViewFlowLayout)
- 날씨 이미지 아이콘을 캐싱하는 Cache 및 FileManager 구현
- CollectionView의 Compositional Layout + Diffable DataSource 구현


<br/>

# 트러블 슈팅
###  범용성 있는 네트워크 통신
**[문제]**
- 이미지를 가져오는 네트워크 통신과 api서버의 json 데이터를 decode하는 네트워크 통신이 따로 구현해야 하는 문제.
```swift

// WeatherNetworkManager.swift
final class WeatherNetworkManager {
    
    ...
    
    // URLSession을 통해 네트워크 통신하는 역할
    func loadWeatherData(type: WeatherType, coord: CLLocationCoordinate2D) {
        weatherType = type
        do {
            let request = try WeatherApiClient.makeRequest(weatherType: type, coord: coord)
            let task = session.dataTask(with: request)
            task.resume()
        } catch {
            print(error)
        }
    }
}

// json 데이터를 decode하는 역할
extension WeatherNetworkManager: URLSessionDataDelegate {
    
    func urlSession(_ session: URLSession, task: URLSessionTask, didCompleteWithError error: Error?) {
        
        ...
        
        do {
            let data = try JSONDecoder().decode(weatherType.model, from: receivedData)
            weatherDelegate?.weather(self, didLoad: data)
        } catch {
            print(error)
        }
    }
}
```


**[해결]**
- 여러 책임을 지고 있는 WeatherNetworkManger를 역할별로 클래스로 분리.
--> WeatherNetworkManger는 URLSession을 통해 네트워크 통신을 하는 역할만 남김. 각 데이터를 포맷에 맞게 변환하는 DataService로 분리.
--> json 데이터를 decode하는 클래스(TodayDataService, ForecastDataService), 이미지로 변환하는 클래스(IconDataService)
```swift

// WeatherNetworkManager.swift
final class WeatherNetworkManager {
    func downloadData(url: URL, completionHandler: @escaping (Result<Data, Error>) -> Void) {
        let request = URLRequest(url: url)
        URLSession.shared.dataTask(with: request) { data, response, error in
                                                   
            ...
                                                   
            guard let data = data else {
                return completionHandler(.failure(NetworkError.failedToLoadData))
            }
            
            completionHandler(.success(data))
        }.resume()
    }
}
// ForecastDataService.swift 
final class ForecastDataService: DataServiceable {
    weak var delegate: ForecastDataServiceDelegate?
    
    // json 데이터를 Model에 맞게 decode 하는 역할
    func downloadData(type service: ServiceType) throws {
        
        ...
        
        do {
            let decodeData = try self.decoder.decode(WeatherForecast.self, from: data)
            self.delegate?.forecastDataService(self, didDownload: decodeData)
        } catch {
            print(error)
        }
    }
}

// IconDataService.swift
final class IconDataService: DataServiceable {
    
    // 받아온 데이터를 image로 변환하는 역할
    func downloadData(type service: ServiceType) throws {
        
        ...
        
        guard let image = UIImage(data: data) else {
            print("이미지 변환에 실패했습니다.")
            return
        }
    }
}
```


<br/><br/>

# 프로젝트를 통해 배운점
- CollectionView 사용 및 Compositional layout과 Diffable DataSource 적용
- URLSession을 사용하여 open weather api에서 데이터를 네트워크 통신
- FileManager, Cache를 사용해 이미지를 캐싱하여 사용
