# Use Dynamic Binary Packet for JMeter 
성능 테스트를 위해 가장 많이 사용된다는 Tool인 JMeter을 사용하기로 했습니다.
JMeter는 HTTP, HTTPS, FTP, REST, TCP등 다양한 프로토콜을 지원한다고 하는데, TCP Binary 패킷에 대한 송수신 처리 자료는 찾기 힘들었습니다.
여기서 JMeter를 사용한 가변길이 TCP Binary 패킷 처리를 설명합니다.
JMeter의 beanShell을 사용해서 송수신 패킷을 만들고, 별도 제작한 plugin을 사용해 TCP 송수신 처리를 할 것입니다.

## Test 시나리오
- 테스트할 서버는 로컬에 1234 port를 listen하고 있습니다.
- 요청 패킷에 요청 seq가 포함됩니다.
- 요청/응답 header 패킷은 동일합니다.
```c++
struct header_ {
    unsigned char version;      // 1 byte
    short int seq;              // 2 byte
    unsigned char str;          // 1 byte
    short int type;             // 2 byte
    short int bodyLen;          // 2 byte
}
```
- body는 일반 문자열입니다.
  - 요청 패킷 전문은 CSV 파일을 읽어 내용을 만듭니다.
- thread(socket connection) 는 10개로 합니다.

## apache JMeter 소스 받기
> JMeter의 sampler에서 tcp 가변길이 패킷을 처리할 수 있는 방법이 없으므로, plug-in을 만들어야 합니다. 
> Java Request plug-in을 만들기 위해 jmeter 소스를 받습니다.

#### 1. github 에서 fork
> https://github.com/apache/jmeter
#### 2. 내 PC에 checkout
```bash
git clone https://github.com/lmk/jmeter
```
#### 3. branch 생성
```bash
git branch DynamicLengthTCP
```
#### 4. Eclipse import 준비
```bash
ant setup-eclipse-project
```
#### 5. Eclipse > File > Import > Existing Projects into Workspace

## 코드 추가
#### 1. org.apache.jmeter.protocol.java.test 아래 DynamicLengthTCP, SingletonSocketMap class 추가
  - [DynamicLengthTCP](https://github.com/lmk/jmeter/blob/DynamicLengthTCP/src/protocol/java/org/apache/jmeter/protocol/java/test/DynamicLengthTCP.java)
    - [AbstractJavaSamplerClient](https://github.com/lmk/jmeter/blob/DynamicLengthTCP/src/protocol/java/org/apache/jmeter/protocol/java/sampler/AbstractJavaSamplerClient.java) 상속받아 구현했습니다.
    - 자세한 내용은 [JavaTest](https://github.com/lmk/jmeter/blob/DynamicLengthTCP/src/protocol/java/org/apache/jmeter/protocol/java/test/JavaTest.java) class 참고했습니다. (JavaTest를 그대로 복사해서 생성)
    - 접속할 서버 IP, 포트 등 plug-in에 옵션이 필요한데, [getDefaultParamters()](https://github.com/lmk/jmeter/blob/DynamicLengthTCP/src/protocol/java/org/apache/jmeter/protocol/java/test/DynamicLengthTCP.java#L104)에 정의하면 JMeter에서 GUI까지 만들어 줍니다.
    - Jmeter에서 생성한 Binary 패킷을 plug-in으로 가져오기 위해서는 hex string으로 변환해서 "RequestData" 변수로 넘어옵니다. ([BinaryTCPClientImpl](https://github.com/lmk/jmeter/blob/DynamicLengthTCP/src/protocol/tcp/org/apache/jmeter/protocol/tcp/sampler/BinaryTCPClientImpl.java) 참고)
    - 저장된 요청 패킷을 byte[]로 변환하고([여기](https://github.com/lmk/jmeter/blob/DynamicLengthTCP/src/protocol/java/org/apache/jmeter/protocol/java/test/DynamicLengthTCP.java#L181))  패킷을 전송합니다.
    - 응답 패킷 header 먼저 받고, body 길이만큼 응답 패킷을 받습니다.
    - 받은 Binary 패킷과 상태 정보를, jmeter에서 받을 수 있도록 SampleResult에 담습니다. 
  - [SingletonSocketMap](https://github.com/lmk/jmeter/blob/DynamicLengthTCP/src/protocol/java/org/apache/jmeter/protocol/java/test/SingletonSocketMap.java)
    - Thread별 Socket 재사용(Re-use Connected socket)을 위한 class 입니다.

#### 2. git add 및 commit, push

## Export jar
#### 1. Package Explorer 에서 추가한 파일 선택
  - DynamicLengthTCP.java
  - SingletonSocketMap.java
#### 2. Export > JAR File
#### 3. \lib\ext 경로 아래 저장할 jar 파일명 입력
  - \lib\ext에 jar 파일을 넣어두면, JMeter 실행시 plugin으로 인식해서 읽어들입니다.
  - github에서 받은 소스의 \lib\ext 경로가 아니라, JMeter 실행파일이 위치한 경로입니다.
  - 제 경우는 소스는 "D:\src\jmeter"에 받았고, JMeter 실행파일은 "D:\Tools\apache-jmeter-5.0"에 받았으므로, jar 파일을 저장할 경로는 "D:\Tools\apache-jmeter-5.0\lib\ext" 였습니다.
  
## Test
#### 1. bin/jmeter.bat 실행
#### 2. "Test Plan" 우클릭 > Add > Thread Group추가 
    - Number of Threads를 10 으로 수정합니다.
    - Loop Count는 Forver 체크하면 Stop 할때까지 반복하지만, 테스트니까 기본값 1로 유지합니다.
#### 3. "Thread Group" 우클릭 > Add > Config Element > Counter 추가.
  - 요청 패킷 seq 증가를 위한 것입니다.
  - 시작값, 증가값은 1로 설정합니다.
  - seq 는 패킷의 자료형 크기보다 작게 max 값을 정합니다.
    - 여기서는 short 형이므로 30000으로 잡았습니다.
  - Exported Variable Name에 jmeter 내부에서 사용할 변수명 "reqCount"를 입력합니다.
#### 4. "Thread Group" 우클릭 > Add > Config Element > CSV Data Set Config 추가.
  - Filename에 "msg.txt"로 지정합니다.
    - msg.txt 파일 내용은 아와 같습니다.
    ```text
    메시지1
    메시지2
    메시지3
    ```
    - 여기서는 row전체를 하나의 field를 사용합니다. 
  - Variable Names에 JMeter 내부에서 사용할 변수명 "msg"를 입력합니다.
  - Recyle on EOF: row가 끝나면 처음부터 다시 읽도록 True로 설정합니다.
  - Stop thread on EOF: row가 끝나도 테스트를 계속 할것이므로 False로 설정합니다.
#### 5. 테스트 결과를 쉽게 보기 위해 Listener 추가
  - "Thread Group" 우클릭 > Add > Listener > View Results Tree
  - "Thread Group" 우클릭 > Add > Listener > Summary Report
  - "Thread Group" 우클릭 > Add > Listener > Response Time Graph
#### 6. "Thread Group" 우클릭 > Add > Sampler > Java Request 
  - Classname은 이번에 만든 DynamicLengthTCP 선택합니다.
  - HostIP: 접속할 서버 IP 127.0.0.1 입력합니다.
  - Port: 접속할 서버 Port 1234 입력합니다.
  - Connect Timeout: 접속할때 처리할 timeout 시간은 대략 200 입력합니다.
  - Response Timeout: 응답 패킷을 받을때 timeout 시간은 대략 200 입력합니다.
  - Re-use connect: 하나의 thread에서 connect socket을 재사용할지 여부인데, True를 입력합니다.
  - Set NoDelay, SO_LINGER는 socket 옵션 참고합니다. (모르겠으면 false, 0)
  - Size of response header: 응답 패킷의 해더 사이즈를 입력합니다 (여기서는 8)
  - Offset of response body length: 응답 해더에서 body 사이즈가 포함된 위치를 입력합니다 (여기서는 6) 
  - Size of response body length: body 사이즈가 저장된 크기를 입력합니다 (여기서는 2)
    - DynamicLengthTCP 에서는 사이즈가 1, 2, 4 일때만 처리 가능합니다.
#### 7. "Java Request" 우클릭 > Add > Pre Processors > BeanShell PreProcessor
  - 요청 패킷을 만들기 위해 BeanShell을 작성합니다.
  ```java
import java.io.*;
import org.apache.jmeter.protocol.tcp.sampler.*;

try {
	// msg from CSV Data Set Config
	String str = vars.get("msg");
	byte[] msg = str.getBytes();

	byte[] pkt = new byte[8 + msg.length];
	
	// version
	pkt[0] = (byte)0x01;
	
	// seq from Counter
	short seq = Short.parseShort(vars.get("reqCount"));
	pkt[1] = (byte)(seq>>>8);
	pkt[2] = (byte)seq;
	
	// constString
	pkt[3] = (byte)0x00;
	
	// type
	pkt[4] = (byte)0x00;
	pkt[5] = (byte)0x01;
	
	// len
	pkt[6] = (byte)(msg.length>>>8);
	pkt[7] = (byte)msg.length;
	
	// msg
	System.arraycopy(msg, 0, pkt, 8, msg.length);
	
	// byte array to hex string
	char[] hexArray = "0123456789ABCDEF".toCharArray();
	char[] hexChars = new char[pkt.length *2];
	for(int i=0; i<pkt.length; i++) {
		int v = pkt[i] & 0xff;
		hexChars[i * 2] = hexArray[v>>>4];
		hexChars[i * 2 + 1] = hexArray[v & 0x0f];
	}
	
	str = new String(hexChars);
	vars.put("RequestData", str);
	
	log.info("Request data size: " + str.length());
}catch(Exception e) {
	log.info("Detect BeamShell PreProcessor Exception"+ e);
	prev.setTopThread(true);
}

  ```
#### 8. "Java Request" 우클릭 > Add > Pre Processors > BeanShell PreProcessor
  - 응답패킷 파싱을 위해 BeanShell을 작성합니다.
  ```java
import java.nio.ByteBuffer;
import org.apache.jmeter.samplers.SampleResult;

byte[] resData = prev.getResponseData();
int resSize = resData.length;

byte[] buf1 = {resData[4], resData[5]};
byte[] buf2 = {resData[6], resData[7]};

short type = ByteBuffer.wrap(buf1).getShort();
short bodySize = ByteBuffer.wrap(buf2).getShort();

log.info("Rsponse Size: " + resSize +  ", type: " + type+ ", bodySize: " + bodySize);

  ```

 #### 9. Run > Start
   - 테스트 및 결과 확인합니다.
   
