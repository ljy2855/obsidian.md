
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
![[Pasted image 20240625131400.png]]

### 구현
#### Application

##### Stock
```Java
class Stock implements Parcelable {
    private String name;
    private double previousPrice;
    private double currentPrice;
    private double riskLevel;

    public Stock(String name, double previousPrice, double currentPrice, double riskLevel) {
        this.name = name;
        this.previousPrice = previousPrice;
        this.currentPrice = currentPrice;
        this.riskLevel = riskLevel;
    }

    protected Stock(Parcel in) {
        name = in.readString();
        previousPrice = in.readDouble();
        currentPrice = in.readDouble();
        riskLevel = in.readDouble();
    }

    public static final Creator<Stock> CREATOR = new Creator<Stock>() {
        @Override
        public Stock createFromParcel(Parcel in) {
            return new Stock(in);
        }

        @Override
        public Stock[] newArray(int size) {
            return new Stock[size];
        }
    };

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(name);
        dest.writeDouble(previousPrice);
        dest.writeDouble(currentPrice);
        dest.writeDouble(riskLevel);
    }

    public void updatePrice() {
        final Random random = new Random();
        double changePercent = (random.nextDouble() * 2 * riskLevel) - riskLevel; // Random change within +/- risk level
        double changeAmount = currentPrice * changePercent;
        setCurrentPrice(currentPrice + changeAmount);
    }

    public void setCurrentPrice(double newPrice) {
        this.previousPrice = this.currentPrice;
        this.currentPrice = newPrice;
    }

    public String getName() {
        return name;
    }

    public double getPreviousPrice() {
        return previousPrice;
    }

    public double getCurrentPrice() {
        return currentPrice;
    }

    public String getChange() {
        double change = (currentPrice - previousPrice) / previousPrice * 100;
        return String.format("%.2f%%", change);
    }
}
```
주식 데이터를 저장하기 위한 클래스를 선언한다. 해당 클래스는 안드로이드 sdk에 내장된 `Parcelable` 인터페이스를 상속하여 intent를 통해 데이터 전송시, 해당 클래스로 생성된 instance 객체를 serialize하여 보낼 수 있도록 한다. 
메소드로 주식의 가격을 변경 및 조회하는 기능을 구현한다.

##### UserAccount
```Java
public class UserAccount implements Parcelable {
    private double balance;
    private static UserAccount instance;
    private Map<String, Integer> ownedStocks;

    private UserAccount(double initialBalance) {
        this.balance = initialBalance;
        this.ownedStocks = new HashMap<String, Integer>();
    }

    // 초기화와 함께 인스턴스 생성
    public static synchronized UserAccount initialize(double initialBalance) {
        if (instance == null) {
            instance = new UserAccount(initialBalance);
        }
        return instance;
    }

    // 기존 인스턴스 반환
    public static UserAccount getInstance() {
        if (instance == null) {
            throw new IllegalStateException("UserAccount is not initialized, call initialize(double) first.");
        }
        return instance;
    }

    protected UserAccount(Parcel in) {
        balance = in.readDouble();
        int size = in.readInt();
        ownedStocks = new HashMap<String, Integer>(size);
        for (int i = 0; i < size; i++) {
            String key = in.readString();
            Integer value = in.readInt();
            ownedStocks.put(key, value);
        }
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeDouble(balance);
        dest.writeInt(ownedStocks.size());
        for (Map.Entry<String, Integer> entry : ownedStocks.entrySet()) {
            dest.writeString(entry.getKey());
            dest.writeInt(entry.getValue());
        }
    }

    @Override
    public int describeContents() {
        return 0;
    }

    public static final Creator<UserAccount> CREATOR = new Creator<UserAccount>() {
        @Override
        public UserAccount createFromParcel(Parcel in) {
            return new UserAccount(in);
        }

        @Override
        public UserAccount[] newArray(int size) {
            return new UserAccount[size];
        }
    };

    // Getter and setter methods
    public double getBalance() {
        return balance;
    }

    public Map<String, Integer> getOwnedStocks() {
        return ownedStocks;
    }

    // Additional methods
    public boolean buyStock(Stock stock, int quantity) {
        double totalCost = stock.getCurrentPrice() * quantity;
        if (balance >= totalCost) { // 잔액 확인
            balance -= totalCost; // 잔액 감소
            Integer currentQuantity = ownedStocks.get(stock.getName());
            ownedStocks.put(stock.getName(), (currentQuantity == null ? 0 : currentQuantity) + quantity); // 주식 수량 증가
            return true;
        }
        return false; // 잔액 부족 시 거래 불가
    }

    // 주식 매도
    public boolean sellStock(Stock stock, int quantity) {
        Integer currentQuantity = ownedStocks.get(stock.getName());
        if (currentQuantity != null && currentQuantity >= quantity) { // 보유 주식 수 확인
            balance += stock.getCurrentPrice() * quantity; // 잔액 증가
            int newQuantity = currentQuantity - quantity;
            if (newQuantity > 0) {
                ownedStocks.put(stock.getName(), newQuantity); // 주식 수량 감소
            } else {
                ownedStocks.remove(stock.getName()); // 주식 전부 매도 시 제거
            }
            return true;
        }
        return false; // 보유 주식 수 부족 시 거래 불가
    }
}
```
사용자의 보유주식, 잔고를 저장하기 위한 클래스를 선언한다. Stock과 마찬가지로 `Parcelable`을 사용하여 Intent 전송시, 직렬화가 쉽도록 구현한다. 추가적으로 싱글톤을 적용하여 해당 인스턴스는 단일로만 존재하고 Main과 Detail에서 접근시에도 동일한 객체로 접근하도록 구현하여 데이터의 일관성을 유지할 수 있도록 한다.
주식을 매매하는 기능을 메소드로 구현해 로직와 데이터을 캡슐화 한다. 데이터 접근은 `getOwnedStocks`, `getBalance` 을 통해서만 진행한다.

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
onCreate 시에 `Stock`을 추가하고 리스트뷰에 해당 주식의 가격과 변동폭을 보여주도록 한다. 해당 리스트뷰의 아이템을 클릭 시에 `StockDetailActivity`로 넘어가게 구현한다. 
`priceUpdateService`를 실행시키고 해당 서비스로부터 변경된 주식의 가격을 설정하는 listener를 세팅한다. 주식 가격이 변경되면 UI에 반영을 진행한다.

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

##### Stock Detail Activity

![[Pasted image 20240625131934.png]]
메인 화면에서 주식을 클릭했을 때 보여지는 상세 페이지이다. 해당 화면에서는 메인페이지와 마찬가지로 현재 주식의 가격 및 변동량, 현재 잔고에 추가로 주식 매매가 가능하다.

onCreate
```Java
@Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_stock_detail);

        // Initialize TextViews
        stockNameTextView = (TextView) findViewById(R.id.stockNameTextView);
        previousPriceTextView = (TextView) findViewById(R.id.previousPriceTextView);
        currentPriceTextView = (TextView) findViewById(R.id.currentPriceTextView);
        stockChangeTextView = (TextView) findViewById(R.id.stockChangeTextView);
        balanceTextView = (TextView) findViewById(R.id.balanceTextViewDetail);
        totalAssetsTextView = (TextView) findViewById(R.id.totalAssetsTextViewDetail);
        ownedStocksListView = (ListView) findViewById(R.id.ownedStocksListView);
        // Retrieve the Stock object passed from MainActivity
        stock = getIntent().getParcelableExtra("STOCK");
        userAccount = UserAccount.getInstance();
        updateUI(); // Initial UI update with stock details

        // Define and register BroadcastReceiver to receive price updates
        priceUpdateReceiver = new BroadcastReceiver() {
            @Override
            public void onReceive(Context context, Intent intent) {
                if (StockPriceUpdateService.ACTION_UPDATE_STOCK_PRICE.equals(intent.getAction())) {
                    String stockName = intent.getStringExtra("STOCK_NAME");
                    double newPrice = intent.getDoubleExtra("NEW_PRICE", 0);
                    if (stock.getName().equals(stockName)) {
                        stock.setCurrentPrice(newPrice);
                        updateUI(); // Update the UI with the new stock price
                    }
                }
            }
        };

        // Register the BroadcastReceiver to listen for specific broadcast actions
        registerReceiver(priceUpdateReceiver, new IntentFilter(StockPriceUpdateService.ACTION_UPDATE_STOCK_PRICE));

        Button buyButton = (Button) findViewById(R.id.buttonBuy);
        Button sellButton = (Button) findViewById(R.id.buttonSell);

        buyButton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                if (userAccount.buyStock(stock, 1)) { 
                    updateUI();
                    driver.printTransactionEvent();

                }
            }
        });

        sellButton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                if (userAccount.sellStock(stock, 1)) { 
                    updateUI();
                    driver.printTransactionEvent();
                }
            }
        });
    }
```

`MainActivity`와 마찬가지로 `BroadcastRecevier`를 통해 변경된 주식의 가격을 전달 받는다. 변경사항이 있다면 UI를 업데이트해준다. 추가로 Buy, Sell button에 대한 listener를 설정하여 버튼 클릭시 발생하는 이벤트 핸들러를 구현한다.


updateUI
```Java
private void updateUI() {
        stockNameTextView.setText(stock.getName());
        previousPriceTextView.setText(String.format("Previous Price: $%.2f", stock.getPreviousPrice()));
        currentPriceTextView.setText(String.format("Current Price: $%.2f", stock.getCurrentPrice()));
        double changePercent = (stock.getCurrentPrice() - stock.getPreviousPrice()) / stock.getPreviousPrice() * 100;
        stockChangeTextView.setText(String.format("Change: %.2f%%", changePercent));
        userAccount = UserAccount.getInstance();
        double balance = userAccount.getBalance();
        balanceTextView.setText(String.format("Current Balance: $%.2f", balance));

        double totalAssets = balance;
        Map<String, Integer> stocksOwned = userAccount.getOwnedStocks();

        for (Map.Entry<String, Integer> entry : stocksOwned.entrySet()) {

            totalAssets += entry.getValue() * stock.getCurrentPrice(); 
        }
        // 현재 보유 주식 업데이트
        ArrayList<Map.Entry<String, Integer>> ownedStocksList = new ArrayList<Map.Entry<String, Integer>>(
                stocksOwned.entrySet());
        if (ownershipAdapter == null) {
            ownershipAdapter = new StockOwnershipAdapter(this, ownedStocksList);
            ownedStocksListView.setAdapter(ownershipAdapter);
        } else {
            ownershipAdapter.clear();
            ownershipAdapter.addAll(ownedStocksList);
            ownershipAdapter.notifyDataSetChanged();
        }

        totalAssetsTextView.setText(String.format("Total Assets: $%.2f", totalAssets));

        Integer quantity = stocksOwned.get(stock.getName());
        if (quantity == null) {
            quantity = 0;
        }
        driver.printOwnedStockQuantity(quantity);

        // Assuming 'isSell' is a valid boolean variable that indicates if the last
        // transaction was a sell
        driver.printTransactionMode(isSell);
        driver.printAccount(totalAssets, balance);

    }
```
Main과 마찬가지로 주식 가격에 대한 변동을 반영하고 추가적으로 보유한 주식에 대한 변경도 적용한다. 먼저 UserAccount 인스턴스로부터 잔고 및 보유 주식을 가져오고 총 자산에 대한 계산을 진행한 뒤 UI에 반영한다.


#### JNI
```Java
public class JniDriver {
    static {
        System.loadLibrary("jni_driver");
    }

    public void clearDevice() {
        this.printDOT("");
        this.printFND("0000");
        this.printLCD("");
        this.printLED(0);
    }

    public void printAccount(double total, double balance) {
        // Format the strings to fit within the LCD's display lines
        String firstLine = String.format("Total:  $%.2f", total);
        String secondLine = String.format("Balance: $%.2f", balance);

        // Ensure each line does not exceed 16 characters
        if (firstLine.length() > 16) {
            firstLine = firstLine.substring(0, 16);
        }
        if (secondLine.length() > 16) {
            secondLine = secondLine.substring(0, 16);
        }

        // Concatenate the two lines with enough spaces to fill each line if necessary
        String fullMessage = String.format("%-16s%-16s", firstLine, secondLine);

        // Send the formatted string to the LCD display
        this.printLCD(fullMessage);
    }

    public void printTransactionMode(boolean isSell) {
        if (isSell) {
            this.printDOT("S");
        } else {
            this.printDOT("B");
        }
    }

    public void printOwnedStockQuantity(Integer quantity) {
        String str = String.valueOf(quantity); 
        this.printFND(str);
    }

    public void printPriceChangeEvent() {
        this.printLED(255);
        try {
            Thread.sleep(1000); 
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        this.printLED(0);
    }

    public void printTransactionEvent() {
        this.runMotor(1, 1, 1, 1);
    }

    public native int readKey();

    // 네이티브 메소드 선언
    private native void printLCD(String text);

    private native void printDOT(String text);

    private native void printFND(String text);

    private native void printLED(int num);

    private native void runMotor(int action, int direction, int speed, int time);

}
```
FPGA device를 제어하기 위하여 JNI를 통해서 디바이스 드라이버를 제어한다. 우선 클래스를 선언하여 사용할 JNI method들을 선언한다.

사용할 FPGA 디바이스 별 기능은 아래와 같다.
* LCD : 현재 총 자산 및 보유 잔고 출력
* DOT : 주식 매매 모드 출력 (Buy, Sell)
* FND : 개별 주식의 보유량 출력
* LED : 주식 가격 변동시 1초간 점등
* MOTOR : 주식 거래시에 1초간 회전
* RESET, HOME button : 각각 주식 거래 버튼, 거래 모드 변경

![[Pasted image 20240625135348.png]]

해당 Jni function을 통해 진행할 io 작업에 대한 method들을 추가로 구현한다.


**Complie**
```sh
#!/bin/bash

# Java 소스 파일 위치 설정
JAVA_SRC_PATH="./src"
# JNI 헤더가 저장될 폴더 설정
JNI_HEADER_PATH="./jni"

# 현재 디렉토리 저장
CURRENT_DIR=$(pwd)

# Java 클래스 파일을 컴파일하기 위해 src 폴더로 이동
cd $JAVA_SRC_PATH

# Java 파일 컴파일 (클래스 파일 생성)
javac com/example/FinalProj/JniDriver.java
if [ $? -ne 0 ]; then
    echo "Java compilation failed!"
    exit 1
fi

# JNI 헤더 파일 생성
javah -jni com.example.FinalProj.JniDriver
if [ $? -ne 0 ]; then
    echo "JNI header generation failed!"
    exit 1
fi

# 생성된 헤더 파일을 JNI 폴더로 이동
mv com_example_FinalProj_JniDriver.h $CURRENT_DIR/$JNI_HEADER_PATH/

# 원래 폴더로 돌아가기
cd $CURRENT_DIR

echo "JNI header generated and moved successfully."
```
해당 클래스를 컴파일 후 c header 파일을 생성하는 스크립트를 만들어 자동화 시킨다.

**JniDriver.h**
```C
/* DO NOT EDIT THIS FILE - it is machine generated */
#include <jni.h>
/* Header for class com_example_FinalProj_JniDriver */

#ifndef _Included_com_example_FinalProj_JniDriver
#define _Included_com_example_FinalProj_JniDriver
#ifdef __cplusplus
extern "C" {
#endif
/*
 * Class:     com_example_FinalProj_JniDriver
 * Method:    readKey
 * Signature: ()I
 */
JNIEXPORT jint JNICALL Java_com_example_FinalProj_JniDriver_readKey
  (JNIEnv *, jobject);

/*
 * Class:     com_example_FinalProj_JniDriver
 * Method:    printLCD
 * Signature: (Ljava/lang/String;)V
 */
JNIEXPORT void JNICALL Java_com_example_FinalProj_JniDriver_printLCD
  (JNIEnv *, jobject, jstring);

/*
 * Class:     com_example_FinalProj_JniDriver
 * Method:    printDOT
 * Signature: (Ljava/lang/String;)V
 */
JNIEXPORT void JNICALL Java_com_example_FinalProj_JniDriver_printDOT
  (JNIEnv *, jobject, jstring);

/*
 * Class:     com_example_FinalProj_JniDriver
 * Method:    printFND
 * Signature: (Ljava/lang/String;)V
 */
JNIEXPORT void JNICALL Java_com_example_FinalProj_JniDriver_printFND
  (JNIEnv *, jobject, jstring);

/*
 * Class:     com_example_FinalProj_JniDriver
 * Method:    printLED
 * Signature: (I)V
 */
JNIEXPORT void JNICALL Java_com_example_FinalProj_JniDriver_printLED
  (JNIEnv *, jobject, jint);

/*
 * Class:     com_example_FinalProj_JniDriver
 * Method:    runMotor
 * Signature: (IIII)V
 */
JNIEXPORT void JNICALL Java_com_example_FinalProj_JniDriver_runMotor
  (JNIEnv *, jobject, jint, jint, jint, jint);

#ifdef __cplusplus
}
#endif
#endif

```
생성된 JNI function들을 참고하여 드라이버를 제어하는 로직을 추가한다.

jni_driver.c
```C
#include "com_example_FinalProj_JniDriver.h"
#include <stdio.h>
#include "android/log.h"
#include "output_driver.h"
#include "input_driver.h"

#define LOG_TAG "MyTag"
#define LOGV(...) __android_log_print(ANDROID_LOG_VERBOSE, LOG_TAG, __VA_ARGS__)

JNIEXPORT void JNICALL Java_com_example_FinalProj_JniDriver_printLCD(JNIEnv *env, jobject obj, jstring text)
{
    const char *nativeText = (*env)->GetStringUTFChars(env, text, 0);
    if (nativeText == NULL)
    {
        return; // GetStringUTFChars failed to allocate memory
    }
    LOGV("Print to LCD: %s", nativeText);
    print_to_lcd(nativeText);
    (*env)->ReleaseStringUTFChars(env, text, nativeText);
}

JNIEXPORT void JNICALL Java_com_example_FinalProj_JniDriver_printDOT(JNIEnv *env, jobject obj, jstring text)
{
    const char *nativeString = (*env)->GetStringUTFChars(env, text, 0);
    if (nativeString == NULL)
    {
        return; // GetStringUTFChars failed to allocate memory
    }
    LOGV("Print to DOT Matrix: %s", nativeString);
    print_dot(nativeString);
    (*env)->ReleaseStringUTFChars(env, text, nativeString);
}

JNIEXPORT void JNICALL Java_com_example_FinalProj_JniDriver_printFND(JNIEnv *env, jobject obj, jstring str)
{
    const char *numStr = (*env)->GetStringUTFChars(env, str, NULL);
    if (numStr == NULL)
    {
        return; // GetStringUTFChars failed to allocate memory
    }
    LOGV("Print to FND: %s", numStr);
    print_fnd(numStr);
    (*env)->ReleaseStringUTFChars(env, str, numStr);
}

JNIEXPORT void JNICALL Java_com_example_FinalProj_JniDriver_printLED(JNIEnv *env, jobject obj, jint value)
{
    LOGV("Print to LED: %d", value);
    print_led((unsigned char)value);
}

JNIEXPORT void JNICALL Java_com_example_FinalProj_JniDriver_runMotor(JNIEnv *env, jobject obj, jint action, jint direction, jint speed, jint duration)
{
    LOGV("Run motor - Action: %d, Direction: %d, Speed: %d, Duration: %d", action, direction, speed, duration);
    run_motor(action, direction, speed, duration);
}

JNIEXPORT jint JNICALL Java_com_example_FinalProj_JniDriver_readKey(JNIEnv *env, jobject obj)
{
    int result = 0;
    result = readkey();
    LOGV("Read Key: %d", result);
    return result;
}
```
- printLCD, printDOT, printFND : 인자로 넘겨온 String 객체를 c로 변환하기 위해 `GetStringUTFChars` 사용 후 release
- runMotor, readKey의 인자는 변환없이 jint 타입 그대로 사용후 return

**printLCD**
```C
void print_to_lcd(const char *text)
{
    int dev;
    char buffer[MAX_BUFF];
    int length;

    // 디바이스 파일 열기
    dev = open(FPGA_TEXT_LCD_DEVICE, O_WRONLY);
    if (dev < 0)
    {
        LOGV("Failed to open the LCD device");
        return;
    }

    // 문자열을 LCD 버퍼 크기에 맞게 복사
    memset(buffer, ' ', sizeof(buffer));
    length = strlen(text);
    if (length > MAX_BUFF)
    {
        length = MAX_BUFF;
    }
    strncpy(buffer, text, length);

    // 디바이스로 문자열 쓰기
    write(dev, buffer, MAX_BUFF);
    LOGV("Finish write to LCD");
    // 디바이스 파일 닫기
    close(dev);
}
```

- text를 인자로 받아 디바이스 드라이버를 열고 32 byte의 크기에 맞춰 해당 text를 출력

**printDOT**
```C
unsigned char fpga_alphabet_patterns[2][10] = {
    {// 'S'
     0x3e, 0x7f, 0x60, 0x60, 0x3e, 0x7f, 0x03, 0x03, 0x7f, 0x7e},
    {// 'B'
     0x7e, 0x7f, 0x63, 0x63, 0x7f, 0x7f, 0x63, 0x63, 0x7f, 0x7e}};

unsigned char fpga_set_blank[10] = {
    // memset(array,0x00,sizeof(array));
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00};
    
void print_dot(const char *text)
{
    int dev, i;
    size_t str_size;

    dev = open(FPGA_DOT_DEVICE, O_WRONLY);
    if (dev < 0)
    {
        LOGV("Device open error: %s\n", FPGA_DOT_DEVICE);
        return;
    }

    if (strcmp(text, "S") == 0)
    {
        str_size = sizeof(fpga_alphabet_patterns[0]);
        write(dev, fpga_alphabet_patterns[0], str_size);
    }
    else if (strcmp(text, "B") == 0)
    {
        str_size = sizeof(fpga_alphabet_patterns[1]);
        write(dev, fpga_alphabet_patterns[1], str_size);
    }
    else
    {
        str_size = sizeof(fpga_set_blank);
        write(dev, fpga_set_blank, str_size);
    }

    close(dev);
}
```
- 주식의 매매 모드 'S', 'B'를 출력하기 위한 패턴 저장
- 인자로 넘겨온 text를 확인후 해당 패턴 출력 아니라면 출력 초기화

**printFND**
```C
void print_fnd(const char *text)
{
    int dev, i;
    unsigned char data[MAX_DIGIT] = {0};
    int str_size = strlen(text);

    if (str_size > MAX_DIGIT)
    {
        str_size = MAX_DIGIT;
        LOGV("Warning! 4 Digit number only!\n");
    }

    for (i = 0; i < str_size; i++)
    {
        if (text[i] >= '0' && text[i] <= '9')
            data[3 - i] = text[i] - '0';
    }

    dev = open(FND_DEVICE, O_RDWR);
    if (dev < 0)
    {
        LOGV("Device open error : %s\n", FND_DEVICE);
        return;
    }

    if (write(dev, &data, 4) < 0)
    {
        LOGV("Write Error!\n");
    }

    close(dev);
}
```
- 인자로 받아온 문자열의 길이가 4이하인지 확인
- FND에 출력 가능한 형태로 변환
- 디바이스에 write후 디바이스 파일 close

**printLED**
```C
void print_led(unsigned char data)
{
    int dev;

    if (data < 0 || data > 255)
    {
        LOGV("Invalid range! Value must be between 0 and 255.\n");
        return;
    }

    dev = open(LED_DEVICE, O_RDWR);
    if (dev < 0)
    {
        LOGV("Device open error : %s\n", LED_DEVICE);
        return;
    }

    if (write(dev, &data, 1) < 0)
    {
        LOGV("Write Error!\n");
    }

    close(dev);
}
```
- 넘겨받은 인자의 값 확인 0 - 255
- 디바이스 파일 오픈 후 해당 데이터 write 후 close

**runMotor**
```C
void run_motor(int action, int direction, int speed, int duration)
{
    int dev;
    unsigned char motor_state[3];

    dev = open(FPGA_STEP_MOTOR_DEVICE, O_WRONLY);
    if (dev < 0)
    {
        perror("Failed to open the motor device");
        return;
    }

    motor_state[0] = (unsigned char)action;
    motor_state[1] = (unsigned char)direction;
    motor_state[2] = (unsigned char)speed;

    write(dev, motor_state, 3);
    sleep(duration);

    motor_state[0] = 0;
    motor_state[1] = (unsigned char)direction;
    motor_state[2] = (unsigned char)speed;
    write(dev, motor_state, 3);
    close(dev);
}
```
- 인자를 각각 받아와 디바이스에 write
- 넘겨받은 duration만큼 sleep 이후 정지상태로 만들고 file close

readKey
```C
#define DEVICE_NAME "/dev/new_driver"
int readkey()
{
    int dev, key;
    dev = open(DEVICE_NAME, O_RDONLY);
    if (dev < 0)
    {
        perror("Failed to open the motor device");
        return;
    }
    key = read(dev, "", 1);
    close(dev);
    return key;
}
```
- 구현한 device module file open
- read를 호출하여 입력받은 key 저장
- read가 끝나면 device file close후 key 반환

#### device driver
여러개의 입력을 감지하고 read 시스템콜을 통해 반환하는 driver module을 작성한다. 입력을 받을 스위치는 RESET과 HOME으로 각각 ioremap, interrupt을 통해 받아오게 한다.

![[Pasted image 20240625141258.png]]

```C
static int __init inter_init(void) {
	int result;
    // init intr
	if((result = inter_register_cdev()) < 0 )
		return result;
    home_flag = false;

    init_timer(&(my_timer));
    my_timer.expires = jiffies +(HZ / 10);
    my_timer.function = timer_callback;
    add_timer(&my_timer);

    // init device
    fpga_rst_addr = ioremap(FPGA_RST_BASE_ADDR, 0x1);

	printk(KERN_ALERT "Init Module Success \n");
	printk(KERN_ALERT "Device : /dev/new_driver, Major Num : 242 \n");
	return 0;
}
static int inter_open(struct inode *minode, struct file *mfile){
	int ret;
	int irq;

	printk(KERN_ALERT "Open Module\n");

	// HOME
	gpio_direction_input(IMX_GPIO_NR(1,11));
	irq = gpio_to_irq(IMX_GPIO_NR(1,11));
	printk(KERN_ALERT "IRQ Number : %d\n",irq);
	ret=request_irq(irq, inter_handler, IRQF_TRIGGER_FALLING, "home", 0);

    
	return 0;
}
```

드라이버 init 시에 0.1초 주기의 타이머를 세팅한다. 해당 타이머는 reset버튼이 눌러졌나 확인을 진행한다. 리셋버튼의 상태를 확인하기 위해 ioremap을 통해 물리 메모리를 매핑하고 주소를 저장한다. 
드라이버 open시에는 request_irq를 통해 isr에 home버튼에 해당하는 gpio핀에 해당하는 interrupt handler를 등록한다.

```C
static int inter_read(struct file *filp, const char *buf, size_t count, loff_t *f_pos){
    
    interruptible_sleep_on(&wq);
    if(home_flag){
        home_flag = false;
        return HOME;
    }
    return RESET;
    
}
```
read 호출시에 일단 프로세스를 wait queue에 삽입후 interrupt 또는 timer에서 깨울 수 있도록 sleep을 진행한다. 해당 프로세스는 block된 상태로 cpu 자원을 독점하지 않게 한다.


```C
irqreturn_t inter_handler(int irq, void* dev_id, struct pt_regs* reg) {
	printk(KERN_ALERT "Interrupt: Home button pressed.\n");
    home_flag = 1;
    __wake_up(&wq, 1, 1, NULL);   // Wake up processes in the wait queue
	return IRQ_HANDLED;
}


void timer_callback(struct timer_list *data) {
    printk(KERN_INFO "Timer Callback function called.\n");
    if(inw((unsigned int)fpga_rst_addr) == 0){
        printk(KERN_INFO "Reset button pressed.\n");
        __wake_up(&wq, 1, 1, NULL); 
    
    }
    my_timer.expires = jiffies +(HZ / 10);
    my_timer.function = timer_callback;
    add_timer(&my_timer);
}
```
interrupt handler는 home버튼 press시에 프로세스의 control flow를 빼앗고 해당 interrupt handler를 호출하여 sleep중인 프로세스를 깨우게 된다.
timer handler는 커널상에서 드라이버 insmod부터 주기적으로 0.1초마다 돌며 reset버튼의 상태를 확인하고 wait queue에서 sleep 중인 process를 깨우게 된다.
해당 두개의 handler는 독립적으로 각자의 상태에 영향을 주지 않고 read중인 프로세스의 깨우고 입력한 key를 전달 할 수 있도록 한다.


### 빌드 및 소스코드 
#### 프로젝트 구조
```sh
❯ tree
.
├── AndroidManifest.xml
├── FinalProj.apk
├── README.md
├── compile_jni.sh # complie java src to jni c header script
├── driver
│   ├── module
│   │   ├── Makefile
│   │   ├── new_driver.c
│   │   └── new_driver.ko
│   ├── objs
│   │   ├── fpga_buzzer_driver.ko
│   │   ├── fpga_dip_switch_driver.ko
│   │   ├── fpga_dot_driver.ko
│   │   ├── fpga_fnd_driver.ko
│   │   ├── fpga_insmod.sh
│   │   ├── fpga_led_driver.ko
│   │   ├── fpga_push_switch_driver.ko
│   │   ├── fpga_step_motor_driver.ko
│   │   ├── fpga_text_lcd_driver.ko
│   │   └── new_driver.ko
│   └── push.sh # driver push script
├── ic_launcher-web.png
├── jni
│   ├── Android.mk
│   ├── com_example_FinalProj_JniDriver.h
│   ├── input_driver.c
│   ├── input_driver.h
│   ├── jni_driver.c
│   ├── output_driver.c
│   └── output_driver.h
├── libs
│   └── android-support-v4.jar
├── lint.xml
├── proguard-project.txt
├── project.properties
├── res
│   ├── layout
│   │   ├── activity_main.xml
│   │   ├── activity_stock_detail.xml
│   │   ├── list_item_stock.xml
│   │   └── stock_ownership_item.xml
└── src
    └── com
        └── example
            └── FinalProj
                ├── JniDriver.java
                ├── MainActivity.java
                ├── StockAdapter.java
                ├── StockDetailActivity.java
                ├── StockOwnershipAdapter.java
                ├── StockPriceUpdateService.java
                └── UserAccount.java
```

안드로이드 커널이 보드에 빌드되어 있는 것을 가정
#### board disk mount
`mount -o rw,remount,size=6G /dev/mmcblk0p4 /data`

#### insert module
##### adb connect
`adb start-server`
##### host push driver file to target board
`./push`
##### target board insert module
`cd /data/local/tmp && sh fpga_insmod.sh`

#### install apk (host)
`adb install FinalProj.apk`


### 느낀점 및 개선 사항
어플리케이션부터 JNI, driver module까지 많은 scope의 부분들을 구현하면서 어느 기능을 어디서 처리할지 설계부터 고민이 많았고 그만큼 문제 발생 및 해결을 해나가는 과정들이 많았습니다. 실시간으로 데이터가 변경되고 이를 입출력하는 과정이 많았던 만큼 동기화에서 많은 문제가 발생했고 해결하는 과정에서 리눅스 및 안드로이드 커널에 대한 학습을 병행하면서 진행했습니다. 이러한 과정 속에서 강의 때 학습했던 내용과 추가적으로 직접 구현하는 내용들에 대한 학습 및 구현이 임베디드라는 분야에 대해 이해를 할 수 있는 계기가 되었습니다.

자잘한 동기화 버그(reset button으로 주식 매매시 motor 작동 X) 등이 남았고 초기에 설계했던 기능들보다 프로젝트 사이즈가 작아졌지만 기능 구현시 확장성 및 유연성을 고려하여 추후 개선 여지를 남겨두었습니다. 
