
## 개요
실시간으로 변동하는 주식 가격을 기반으로 모의 투자를 진행하는 안드로이드 어플리케이션 개발
### 개발 내용
#### 어플리케이션
안드로이드 네이티브 어플리케이션을 구현한다. 어플리케이션에서는 주식가격을 주기동안 주식 위험도에 따라 가격을 랜덤하게 변경한다. 사용자의 화면에서는 가격과 변동폭을 실시간으로 보여줄 수 있도록 한다.
개별 주식들을 매매할 수 있도록 한다. 보유중인 현금과 주식들을 저장하고 주식 가격이 변동됨에 따라 총 자산도 변경될 수 있게 한다.
사용자에게 보유중인 자산들을 확인가능한 인터페이스를 제공한다.
#### JNI
주식 매매 및 총 자산을 FPGA device를 통해서 진행하기 위해 이를 연결할 JNI module을 구현한다. 해당 메소들을 처리하기 위한 클래스를 선언하고 해당 클래스의 객체에서 device를 제어 가능하도록 구현한다.
#### device driver
output을 위한 드라이버는 기존에 제공된 device driver를 이용한다. 다만 여러 디바이스로부터 input을 감지받는 모듈을 하나의 device driver에서 수행한다. 입력은 각각의 디바이스에서 io mapping, interrupt를 통해 받아오고 read 함수내에서 해당 입력을 반환하도록 구현한다.

## 프로젝트
### 개발 일정

- 아이디어 기획 및 기능 설계
- App UI 퍼블리싱
- App 모델 정의 (주식, 계좌)
- App 주식 정보 업데이트 로직 작성
- App 주식 매매, 계좌 업데이트 로직 작성
- JNI 모듈 및 인터페이스 구현
- Device Driver 구현
- Test 및 디버깅
- 리팩토링
### 개발 방법
우선 안드로이드 어플리케이션을 구현하여 UI 및 로직을 구현한다. 초기 단계에서는 FPGA device를 대신해 Button과 Text을 통해 주식 정보 업데이트 및 매매를 테스트한다.
이후 JNI를 통하여 output device driver를 통해 정보들을 디바이스에서도 출력 가능하게 구현한다. 마지막으로 input을 감지할 drvier를 구현하여 reset switch, home button등 다른 종류의 입력을 하나의 드라이버에서 처리할 수 있도록 구현후 JNI를 통해여 연결해준다.

### 시스템 구성도
![[Pasted image 20240625012555.png]]

### 구현
#### Application
##### MainActivity
해당 Activity는 메인화면으로  앱 실행시 초기화면을 보여준다.
![[Pasted image 20240625130546.png]]

**OnCreate**
```Java
@Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        driver = new JniDriver();
        driver.clearDevice();

        userAccount = UserAccount.initialize(INITIAL_BALANCE);
        // Initialize the ListView and Stock list
        listView = (ListView) findViewById(R.id.stockListView);
        balanceTextView = (TextView) findViewById(R.id.balanceTextView);
        totalAssetsTextView = (TextView) findViewById(R.id.totalAssetsTextView);
        ownedStocksListView = (ListView) findViewById(R.id.ownedStocksListView);
        stocks = new ArrayList<Stock>();
        stocks.add(new Stock("Apple Inc.", 150.00, 153.25, 0.05));
        stocks.add(new Stock("Google", 1200.50, 1185.75, 0.03));
        stocks.add(new Stock("Amazon", 3100.00, 3150.00, 0.07));
        stocks.add(new Stock("Microsoft", 200.00, 204.20, 0.02));

        // Initialize the adapter and set it to the ListView
        adapter = new StockAdapter(this, stocks);
        listView.setAdapter(adapter);

        // Set an item click listener for the ListView
        listView.setOnItemClickListener(new AdapterView.OnItemClickListener() {
            public void onItemClick(AdapterView<?> parent, View view, int position, long id) {
                Stock selectedStock = stocks.get(position);
                Intent intent = new Intent(MainActivity.this, StockDetailActivity.class);
                intent.putExtra("STOCK", selectedStock);
                startActivity(intent);
            }
        });
        ArrayList<Map.Entry<String, Integer>> ownedStocksList = new ArrayList<Map.Entry<String, Integer>>(
                userAccount.getOwnedStocks().entrySet());
        ownershipAdapter = new StockOwnershipAdapter(this, ownedStocksList);
        ownedStocksListView.setAdapter(ownershipAdapter);

        // Define and register BroadcastReceiver to receive stock price updates
        priceUpdateReceiver = new BroadcastReceiver() {
            @Override
            public void onReceive(Context context, Intent intent) {
                if (StockPriceUpdateService.ACTION_UPDATE_STOCK_PRICE.equals(intent.getAction())) {
                    String stockName = intent.getStringExtra("STOCK_NAME");
                    double newPrice = intent.getDoubleExtra("NEW_PRICE", 0);
                    for (Stock stock : stocks) {
                        if (stock.getName().equals(stockName)) {
                            stock.setCurrentPrice(newPrice); // Update the price of the matched stock
                            break;
                        }
                    }
                    adapter.notifyDataSetChanged(); // Notify the adapter to refresh the list view
                    updateUI();
                }
            }
        };

        // Register the BroadcastReceiver to listen for specific broadcast actions
        registerReceiver(priceUpdateReceiver, new IntentFilter(StockPriceUpdateService.ACTION_UPDATE_STOCK_PRICE));

        // Start the StockPriceUpdateService
        startService(new Intent(this, StockPriceUpdateService.class));
        updateUI();
    }
```
onCreate 시에 Stock을 추가하고 리스트뷰에 해당 주식의 가격과 변동폭을 보여주도록 한다. 해당 리스트뷰의 아이템을 클릭시에 Stock Detail Activity로 넘어가게 구현한다. 
priceUpdateService를 실행시키고 해당 서비스로부터 변경된 주식의 가격을 설정하는 listener를 세팅한다. 주식 가격이 변경되면 UI에 반영을 진행한다.

**updateUI**
```Java
private void updateUI() {
        double balance = userAccount.getBalance(); // Direct access
        double totalAssets = balance;

        for (Map.Entry<String, Integer> entry : userAccount.getOwnedStocks().entrySet()) {
            for (Stock stock : stocks) {
                if (stock.getName().equals(entry.getKey())) {
                    totalAssets += stock.getCurrentPrice() * entry.getValue();
                }
            }
        }

        balanceTextView.setText(String.format("Balance: $%.2f", balance));
        totalAssetsTextView.setText(String.format("Total Assets: $%.2f", totalAssets));
        driver.printAccount(totalAssets, balance);
    }
```

변경되는 UI를 처리하기 위한 함수이다. 주식 가격이 변동거나 초기에 데이터를 통해 UI를 그릴 수 있도록 현재 user account 정보를 통해 계좌정보를 출력한다.

