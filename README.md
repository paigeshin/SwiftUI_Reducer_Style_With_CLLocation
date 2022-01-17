[https://github.com/paigeshin/SwiftUI_Reducer_Style_With_CLLocation](https://github.com/paigeshin/SwiftUI_Reducer_Style_With_CLLocation)

# Store Setting

### Store

```swift
import Foundation

typealias Dispatcher = (Action) -> Void

typealias Reducer<State: ReduxState> = (_ state: State, _ action: Action) -> State
typealias Middleware<StoreState: ReduxState> = (StoreState, Action, @escaping Dispatcher) -> Void

protocol ReduxState { }

struct AppState: ReduxState {
    var restrooms: RestroomState = RestroomState() 
}

struct RestroomState: ReduxState {
    var restrooms: [Restroom] = []
}

class Store<StoreState: ReduxState>: ObservableObject {
    
    var reducer: Reducer<StoreState>
    @Published var state: StoreState
    var middlewares: [Middleware<StoreState>]
    
    init(reducer: @escaping Reducer<StoreState>, state: StoreState,
         middlewares: [Middleware<StoreState>] = []) {
        self.reducer = reducer
        self.state = state
        self.middlewares = middlewares
    }
    
    func dispatch(action: Action) {
        DispatchQueue.main.async {
            self.state = self.reducer(self.state, action)
        }
        
        // run all middlewares
        middlewares.forEach { middleware in
            middleware(state, action, dispatch)
        }
    }
    
}
```

### AppReducer

```swift
import Foundation

func appReducer(_ state: AppState, _ action: Action) -> AppState {
    
    var state = state
    state.restrooms = restroomsReducer(state.restrooms, action)
    return state
}
```

### RestroomReducer

```swift
import Foundation

func restroomsReducer(_ state: RestroomState, _ action: Action) -> RestroomState {
    var state = state
    
    switch action {
        case let action as SetRestroomsAction:
            state.restrooms = action.restrooms
        default:
            break
    }
    
    return state
}
```

### RestroomMiddleware

```swift
import Foundation

func restroomMiddleware() -> Middleware<AppState> {
    
    return { state, action, dispatch in
        
        switch action {
            case let action as FetchRestroomsAction:
                getRestroomsByLatAndLng(action: action, dispatch: dispatch)
            default:
                break
        }
        
    }
    
}

private func getRestroomsByLatAndLng(action: FetchRestroomsAction, dispatch: @escaping Dispatcher) {
    
    Webservice().getRestroomsByLatAndLng(lat: action.latitude, lng: action.longitude) { result in
        switch result {
            case .success(let restrooms):
                if let restrooms = restrooms {
                    dispatch(SetRestroomsAction(restrooms: restrooms))
                }
            case .failure(let error):
                print(error.localizedDescription)
        }
    }
    
}
```

### Actions

```swift
import Foundation

protocol Action { }

struct FetchRestroomsAction: Action {
    let latitude: Double
    let longitude: Double
}

struct SetRestroomsAction: Action {
    let restrooms: [Restroom]
}
```

# Location Manager

```swift
import Foundation
import CoreLocation

class LocationManager: NSObject, ObservableObject {
    
    private let locationManager = CLLocationManager()
    @Published var location: CLLocation? = nil
    
    override init() {
        super.init()
        
        locationManager.delegate = self
        locationManager.desiredAccuracy = kCLLocationAccuracyBest
        locationManager.distanceFilter = kCLDistanceFilterNone
        locationManager.requestAlwaysAuthorization()
        locationManager.startUpdatingLocation()
    }
    
    func updateLocation() {
        locationManager.startUpdatingLocation()
    }
    
}

extension LocationManager: CLLocationManagerDelegate {
    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        guard let location = locations.last else {
            locationManager.stopUpdatingLocation()
            return
        }
        
        DispatchQueue.main.async {
            self.location = location
        }
        
        locationManager.stopUpdatingLocation()
    }
}
```

# App

```swift
import SwiftUI

@main
struct HelloReduxApp: App {
    
    init() {
        configureTheme()
    }
    
    var body: some Scene {
       
        let store = Store(reducer: appReducer, state: AppState(), middlewares: [restroomMiddleware()])
        
        WindowGroup {
            HomeScreen().environmentObject(store)
        }
    }
    
    private func configureTheme() {
        UINavigationBar.appearance().backgroundColor = UIColor(displayP3Red: 44/255, green: 62/255, blue: 80/255, alpha: 1.0)
    }
}
```

# HomeScreen (with combine)

```swift
import SwiftUI
import Combine

struct HomeScreen: View {
    
    @ObservedObject private var locationManager = LocationManager()
    @EnvironmentObject var store: Store<AppState>
    @State private var cancellables: AnyCancellable? = nil
    
    struct Props {
        let restrooms: [Restroom]
        let onFetchRestroomsByLatLng: (Double, Double) -> Void
    }
    
    private func map(state: RestroomState) -> Props {
        Props(restrooms: state.restrooms, onFetchRestroomsByLatLng: { (lat, lng) in
            store.dispatch(action: FetchRestroomsAction(latitude: lat, longitude: lng))
        })
    }
    
    var body: some View {
        
        let props = map(state: store.state.restrooms)
        
        VStack(alignment: .leading) {
            
            HStack {
                EmptyView()
            }.frame(maxWidth: .infinity, maxHeight: 44)
            Spacer()
            
            HStack {
                Text("Restrooms")
                    .foregroundColor(Color.white)
                    .font(.largeTitle)
                Spacer()
                Button(action: {
                    // force the location to be updated...
                    locationManager.updateLocation()
                }) {
                    Image(systemName: "arrow.clockwise.circle")
                        .font(.title)
                        .foregroundColor(Color.white)
                }
            }.padding()
            
            List(props.restrooms, id: \.id) { restroom in
                RestroomCell(restroom: restroom)
            }
            .buttonStyle(PlainButtonStyle())
            
        }.frame(maxWidth: .infinity, maxHeight: .infinity)
            .background(Color(#colorLiteral(red: 0.880972445, green: 0.3729454875, blue: 0.2552506924, alpha: 1)))
            .edgesIgnoringSafeArea(.all)
        
        
            .onAppear {
								// sink returns Publihser
								// canceelables hold publisher 
                self.cancellables = locationManager.$location.sink { location in
                    if let location = location {
                        print(location)
                        props.onFetchRestroomsByLatLng(location.coordinate.latitude, location.coordinate.longitude)
                    }
                }
            }
        
    }
}

struct HomeScreen_Previews: PreviewProvider {
    static var previews: some View {
        let store = Store(reducer: appReducer, state: AppState(), middlewares: [restroomMiddleware()])
        return HomeScreen().environmentObject(store)
    }
}

struct RestroomCell: View {
    
    let restroom: Restroom
    
    var body: some View {
        VStack(alignment: .leading, spacing: 10) {
            HStack {
                Text(restroom.name ?? "Not available")
                    .font(.headline)
                Spacer()
                Text(String(format: "%.2f miles",restroom.distance))
            }.padding([.top], 10)
            
            Text(restroom.address)
                .font(.subheadline)
                .opacity(0.5)
            
            Button("Directions") {
                guard let targetURL = URL(string: "http://maps.apple.com/?address=\(restroom.address.encodeURL() ?? "")") else {
                    return
                }
                if UIApplication.shared.canOpenURL(targetURL) {
                    UIApplication.shared.open(targetURL, options: [:], completionHandler: nil)
                }
            }.font(.caption)
                .foregroundColor(Color.white)
                .padding(6)
                .background(Color(#colorLiteral(red: 0.184266597, green: 0.8003296256, blue: 0.4435204864, alpha: 1)))
                .cornerRadius(6)
            
            Text(restroom.comment ?? "")
                .font(.footnote)
            
            HStack {
                Text(restroom.accessible ? "♿️" : "")
            }
        }
    }
}
```
