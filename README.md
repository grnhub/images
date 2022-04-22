# 커리큘럼 정리


## 진행과정

### 1주차

- 개발환경 구축

### 2주차

- 오라클 성능지표 수집(SQL, Session, DB Stat)
    - 다수의 클라이언트를 처리할 수 있는 TCP/IP 프로토콜을 이용한 Blocking 방식의 소켓 서버 구현
    - 주기적으로 오라클 성능 데이터를 수집하여 서버로 전송하는 클라이언트 구현

### 3주차

- 요청응답 구현
    - 서버가 전달받은 SQL 데이터 중 SQL ID에 해당하는 텍스트가 없다면 클라이언트에 요청하여 수집하고 요청에 대한 Timeout을 고려함
    - 서버 재기동을 하더라도 당일 수집한 SQL ID가 중복으로 수집되지 않도록 하고 하루 단위로 재수집
- Postgres 트리거 기반 파티셔닝 구현

### 4주차

- DB Stat 데이터를 메모리에서 1분동안 평균값, 최대값, 최소값을 Java8의 스트림을 활용해 서머리하여 저장
- DB Stat 1분 서머리 데이터 기반으로 1시간, 1일 서머리 프로시저 작성

### 5주차

- Database Connection Pool 구현
- Logger 구현

### 6주차

- 알람기능 설계 및 구현
    - 수집되는 데이터를 기반으로 설정한 임계치를 넘어서는 값 수집시 알람발생
    - 발생하는 알람 데이터를 잔디로 연동


# 내용

## 2주차

### Socket Server

- TCP
    - TCP 서버의 역할
        1. 클라이언트가 연결요청을 하면 연결을 수락
        2. 연결된 클라이언트와 통신

      → 자바에서는 이 역할을 대신해주는 클래스(Socket)를 따로 제공

      ![소켓서버](https://user-images.githubusercontent.com/48713266/164604037-5dc5ab14-8cd8-4063-97c4-f29f72eb5a70.png)


    - 소켓 통신 구현과정
    
    ![소켓서버_통신과정2](https://user-images.githubusercontent.com/48713266/164604129-9febfd81-ccee-49cd-be9e-c0ded34c3c8e.png)


### JDBC

- JDBC 진행 과정
    1. JDBC 드라이버 로드

       DBMS 별로 맞는 드라이버 필요(ojdbc6-112.0.4.jar, postgresql-9.4.1212.jar)

       Class.forName(”JDBC 드라이버 이름”)

    2. DB 연결

       DriverManager.getConnection(url, id, pwd)

    3. SQL setting

       Statement / PreparedStatement

    4. SQL 실행

       executeQuert(), executeUpdate()

    5. ResultSet(SELECT의 경우)
    6. DB 연결 종료

       Close(Connection, Statement, ResultSet)


### DataInputStream, DataOutputStream

- Primitive Type Data를 읽고 쓰는데 알맞는 클래스
- 각각 생성자에서 InputStream과 OutputStream을 받는다.

    ```
    Socket socket;
    
    InputStream is = socket.getInputStream();
    OutputStream os = socket.getOutputStream();
    
    /* 각 생성자에서 InputStream과 OutputStream을 받음 */
    DataInputStream dis = new DataInputStream(is);
    DataOutputStream dos = new DataOutputStream(os);
    ```

  생성자의 arguments 값으로 InputStream, OutputStream을 받는다는 것은 그 하위 클래스들도 받아들인다는 의미(FileInputStream, FimeOutputStream)


## 3주차

### 요청응답 구현

- 서버가 전달받은 SQL 데이터 중 SQL_ID에 해당하는 텍스트가 없다면 클라이언트에 요청 → 요청받은 클라이언트는 SQL_TEXT를 수집하여 서버에 데이터 전송

![소켓서버_통신과정](https://user-images.githubusercontent.com/48713266/164604134-e57da078-9299-456a-99e3-e52c381ce634.png)

sql id 요청 및 sql text 수집 기능 추가 후 구현 과정

### SQL ID 요청

클라이언트로부터 전달받은 Session Stqt, Sql Stat 데이터 중 sql id에 해당하는 텍스트가 없다면 클라이언트에 요청하기 위해 해당 아이디를 메모리에 저장한다. 이때 서버가 재기동할 경우에도 sql id를 중복으로 수집하지 않도록 한다. **sql id를 저장하는 메모리**는 Server 클래스에 전역적으로 선언하고 **순차적으로 처리하도록 Queue를 사용**했다.

```
//당일 수집된 sqlId 저장 메모리
public static Queue<String> todaySqlIdInMemoryQueue = new LinkedList<>();
//sqlId 정보 저장 메모리
public static BlockingQueue<SqlId> sqlIdInMemoryQueue = new LinkedBlockingQueue<>();
```

```
//sqlId를 메모리에 저장
public static void setSqlIdInMemoryQueue(byte clientId, String sqlId) {
    SqlId sqlIdVO = new SqlId();
    sqlIdVO.setClientId(clientId);
    sqlIdVO.setSqlId(sqlId);

    //당일 수집한 sql id가 아닐 경우 수행
    if(!todaySqlIdInMemoryQueue.contains(sqlId)) {
        sqlIdInMemoryQueue.add(sqlIdVO);
        todaySqlIdInMemoryQueue.add(sqlId);
    }
}
```

### SQL ID 요청에 따른 Timeout

timeout 된 sql id를 저장하는 map을 Server 클래스에 전역적으로 선언하고 timeout 된 sql id를 보여주는 타이머를 주기적으로 실행시킨다.

```
/*
	timeout된 sql id를 저장하는 map
	전역적으로 선언
*/
public static Map<String, SqlId> timeoutSqlIdMap = new HashMap<>();
...

//timeout 된 sql id를 보여줌
Timer sqlIdTimer = new Timer();
SqlIdTimeoutManage timeoutSqlIdTask = new SqlIdTimeoutManage();
sqlIdTimer.schedule(timeoutSqlIdTask, 3000, 10000);
```

```
Timestamp now = new Timestamp(System.currentTimeMillis());

//timeout 된 sql id 요청이 없으면 return
if(timeoutSqlIdMap.isEmpty()) {
		return;
}

for(Map.Entry<String, SqlId> sqlIdEntry : timeoutSqlIdMap.entrySet()) {
    SqlId timeoutMapValue = sqlIdEntry.getValue();
    Timestamp reqTime = new Timestamp(timeoutMapValue.getTime());
    Timestamp stdTime = new Timestamp(reqTime.getTime() + 5000);
    //현재시간이 최대응답시간을 지났으면 Timeout
    if(now.after(stdTime)) {
        uuid = sqlIdEntry.getKey();
        logger.info("[Timeout] SQL ID : " + timeoutMapValue.getSqlId()
                        + ", 요청시간 : " + reqTime
                        + ", 최대응답시간 : " + stdTime
                        + ", 현재시간 : " + now);
        break;
    }
}
```

### Partitioning

- 파티셔닝 테이블을 생성하는 프로시저

```sql
CREATE OR REPLACE FUNCTION public.create_db_stat_partition()
 RETURNS integer
 LANGUAGE plpgsql
AS $function$
DECLARE
	partition_key DATE;
	start_date INTEGER;
	end_date INTEGER;
	i_date DATE;
	i_date_text TEXT;
	partition_name TEXT;
	delete_date DATE;
	delete_partition_name TEXT;
	resultTableCnt INTEGER;
BEGIN
	partition_key := CURRENT_DATE;
	start_date := TO_CHAR(partition_key, 'J')::NUMERIC;
	end_date := TO_CHAR(partition_key + INTERVAL '3 day', 'J')::NUMERIC;
	resultTableCnt := 0;
	FOR i IN start_date..end_date LOOP	--오늘 기준으로 3일치 파티셔닝 자식테이블 생성
		i_date := TO_DATE(TO_CHAR(i, '999999999999999'), 'J');
		i_date_text := TO_CHAR(i_date, 'YYMMDD');
		partition_name := 'ora_db_stat_p' || i_date_text;
		IF NOT EXISTS(SELECT tablename FROM pg_tables WHERE tablename = partition_name) THEN
		EXECUTE 'CREATE TABLE ' || partition_name || ' (CHECK (TO_CHAR(time::DATE, ''YYYY-MM-DD'') = '''|| i_date ||''')) INHERITS (public.ora_db_stat);';
		RAISE NOTICE 'A partition has been created %', partition_name;
		resultTableCnt := resultTableCnt + 1;
		END IF;
	END LOOP;
	delete_date := partition_key - INTERVAL '6 day';
	delete_partition_name := 'ora_db_stat_p' ||  TO_CHAR(delete_date, 'YYMMDD');
	IF EXISTS(SELECT tablename FROM pg_tables WHERE tablename = delete_partition_name) THEN
		EXECUTE 'DROP TABLE ' || delete_partition_name || ';';
	END IF;
RETURN resultTableCnt;
END $function$
;
```

테이블이 생성된 모습

![파티셔닝테이블생성](https://user-images.githubusercontent.com/48713266/164604161-0a89ffd2-6272-4117-a339-c01d37151d9a.png)

- 데이터 입력 트리거

  각 row마다 실행


```sql
create trigger insert_db_stat_trigger before
insert
    on
    public.ora_db_stat for each row execute procedure db_stat_insert_func()
```

- 데이터를 넣는 함수

```sql
CREATE OR REPLACE FUNCTION public.db_stat_insert_func()
 RETURNS trigger
 LANGUAGE plpgsql
AS $function$
DECLARE
	partition_name TEXT;
BEGIN
	partition_name := TG_RELNAME || '_p' || NEW.partition_key;
	EXECUTE 'INSERT INTO ' || partition_name || ' SELECT ( ' || TG_RELNAME || ' ' || quote_literal(NEW) || ').*;';
RETURN NULL;
END;
$function$
;
```

## 4주차

- DB Stat 데이터 서머리 플로우 차트

![dbstat서머리플로우차트](https://user-images.githubusercontent.com/48713266/164604183-a9266831-1b92-4d95-9390-8bd389b423a7.png)

- DB Stat 1시간 서머리 프로시저

```sql
CREATE OR REPLACE FUNCTION public.insert_db_stat_1hour(std_time timestamp without time zone)
 RETURNS void
 LANGUAGE plpgsql
AS $function$
DECLARE
	compare_time TIMESTAMP := DATE_TRUNC('hour', std_time - interval '1 hour'); 
BEGIN
	INSERT INTO ora_db_stat_1hour (partition_key, db_id, time, stat_id, avg_value, max_value, min_value)
		SELECT 	partition_key
				,db_id
				,compare_time
				,stat_id 
				,ROUND(AVG(avg_value))
				,MAX(max_value)
				,MIN(min_value)
		FROM (	SELECT *
				FROM ora_db_stat
				WHERE DATE_TRUNC('hour', TIME) = compare_time
			) a
		GROUP BY db_id, stat_id, partition_key;
END;
$function$
;
```

- Stream으로 평균값, 최대값, 최소값 구하기

```
List<Long> valueList = statValueMap.get(statId);
LongSummaryStatistics summary = valueList.stream().mapToLong(x -> x).summaryStatistics();
summary.getAverage();    //avg_value
summary.getMax();        //max_value
summary.getMin();        //min_value
```

## 5주차

### Database Connection Pool

애플리케이션이 데이터베이스와 상호 작용할 때 연결을 가져오는 것과 연결을 해제할 때 시스템 자원을 많이 소비한다. 이를 개선하기 위해 DBCP(Database Connection Pool)을 활용한다. DBCP의 방식은 미리 설정해 놓은 일정수의 Connection을 만들어 Connection Pool에 보관해두었다가 요청이 발생하면 Connection을 제공하고 사용이 끝나면 다시 Connection Pool에 반환하여 보관한다.

![DBCP](https://user-images.githubusercontent.com/48713266/164604244-46b1093c-cecd-492d-a1f9-c698318dfae4.png)

- 플로우차트 및 코드(src>com>client>connection>OracleConnectionPool.java)

![커넥션풀플로우차트](https://user-images.githubusercontent.com/48713266/164604261-48ca84cf-19d0-4438-9c21-b75cf9c2a7c1.png)

```
/* 최소유지 개수만큼 커넥션 생성 */
public OracleConnectionPool() {
    try {
        Class.forName(DRIVER_CLASS_NAME);
        for(int i = 0; i < minIdle; i++) {
            try {
		            Connection conn = DriverManager.getConnection(URL, USER, PWD);
		            long time = System.currentTimeMillis();
		            connectionPool.add(new ConnectionData(conn, time));
		            logger.debug("커넥션 생성 : " + conn.toString());
		        } catch (SQLException e) {
		            logger.error(e);
		        }
        }
    } catch (ClassNotFoundException e) {
        logger.error(e);
    }
}
```

```
/*
	커넥션호출 요청이 오면 커넥션을 리턴한다.
  이때 리턴할 커넥션이 부족하면 최대생성가능 개수만큼 커넥션 생성
*/
public synchronized ConnectionData getConnection() {
    if(connectionPool.size() < 1) {         //getConnection이 없으면
        if(connCount <= maxActive) {   //최대 커넥션 개수까지 커넥션 생성
            try {
		            Connection conn = DriverManager.getConnection(URL, USER, PWD);
		            long time = System.currentTimeMillis();
		            connectionPool.add(new ConnectionData(conn, time));
		            logger.debug("커넥션 생성 : " + conn.toString());
		        } catch (SQLException e) {
		            logger.error(e);
		        }
        } else {    //최대 커넥션까지 생성되었다면 대기
            while (connectionPool.size() < 1) {
                try {
                    wait();
                } catch (InterruptedException e) {
                    logger.error(e);
                }
            }
        }
    }
    ConnectionData connDataVO = connectionPool.poll();
    connCount++;
    logger.debug("getConnection::남은 커넥션 수 : " + connectionPool.size() + ", 사용중인 커넥션 수 : " + connCount);
    return connDataVO;
}
```

```
/*
	사용이 끝난 커넥션을 풀로 반환
	이때 생성된 커넥션이 최소유지개수보다 많은 상태에서 커넥션 요청이 줄어들면
	오래된 커넥션을 close하여 최소개수로 유지한다.
*/
public synchronized void returnConnection(ConnectionData connDataVO) {
    connCount--;
    long now = System.currentTimeMillis();
    connDataVO.setTime(now);
    connectionPool.add(connDataVO);
    logger.debug("getConnection::남은 커넥션 수 : " + connectionPool.size() + ", 사용중인 커넥션 수 : " + connCount);
    notifyAll();
    deleteConnection();
}

//오래된 커넥션 삭제
public void deleteConnection() {
    //생성된 커넥션이 최소유지 커넥션 수 보다 많다면 제일 오래된 커넥션 close
    while(connectionPool.size() > minIdle) {
        ConnectionData connDataVO = connectionPool.poll();
        try {
            connDataVO.getConn().close();
        } catch (SQLException e) {
            logger.error(e);
        }
    }
}
```

### Logger

![loggerconfiglevel](https://user-images.githubusercontent.com/48713266/164604279-16af9921-190b-4000-adc1-3e20374c2035.png)

💥 error는 Exception이 발생하면 getMessage()를 출력하던 것을 예외클래스를 넘겨서 Stack Trace를 찍도록 수정함

- 수정 전

```
try {
	...
} catch(IOExeption e) {
	logger.error(e.getMessage());
} catch(SQLExceptin e) {
	logger.error(e.getMessage());
}
```

```
public void error(String msg) {
    int code = 500;
    String codeName = "ERROR";
    printLog(msg, code, codeName);
}
```

- 수정 후

```
try {
	...
} catch(IOExeption e) {
	logger.error(e);
} catch(SQLExceptin e) {
	logger.error(e);
}
```

```
public void error(Throwable e) {
    StringBuilder sb = new StringBuilder();
    for(StackTraceElement element : e.getStackTrace()) {
        sb.append(element.toString());
        sb.append("\n");
    }
    String msg = "Exception occurred\n" + sb.toString();
    int code = 500;
    String codeName = "ERROR";
    printLog(msg, code, codeName);
}
```

💡Logging을 System.out으로 하면 안되는 이유

1. 에러/장애 발생 시 추적하기 힘들다.
2. 로그의 내용을 가져오기 어렵다.

현재 시스템의 기본정보를 알 수 없다. 예외 발생시 콘솔창에 출력만 하기 때문에 해당 메시지를 나중에 다시 확인 할 수 없다. 이러한 문제점으로 인해 애플리케이션의 상태를 분석하고 유지보수하기에 어려움이 있다. 파일이나 콘솔에 로그를 남길 경우 내용을 프린트하거나 저장할 때 까지 다음 프린트 부분은 대기해야한다. 그렇게 되면 애플리케이션에서는 대기시간이 발생하여 큰 부하를 줄 수 있다.

## 6주차

잔디 웹훅 연동 참고

1. [https://support.jandi.com/hc/ko/articles/210952203-잔디-커넥트-인커밍-웹훅-Incoming-Webhook-으로-외부-데이터를-잔디-메시지로-수신하기](https://support.jandi.com/hc/ko/articles/210952203-%EC%9E%94%EB%94%94-%EC%BB%A4%EB%84%A5%ED%8A%B8-%EC%9D%B8%EC%BB%A4%EB%B0%8D-%EC%9B%B9%ED%9B%85-Incoming-Webhook-%EC%9C%BC%EB%A1%9C-%EC%99%B8%EB%B6%80-%EB%8D%B0%EC%9D%B4%ED%84%B0%EB%A5%BC-%EC%9E%94%EB%94%94-%EB%A9%94%EC%8B%9C%EC%A7%80%EB%A1%9C-%EC%88%98%EC%8B%A0%ED%95%98%EA%B8%B0)
2. [https://drive.google.com/file/d/0B2qOhquiLKk0TVBqc2JkQmRCMGM/view?resourcekey=0-JmsAKlWbAiygg0sCu-AIcg](https://drive.google.com/file/d/0B2qOhquiLKk0TVBqc2JkQmRCMGM/view?resourcekey=0-JmsAKlWbAiygg0sCu-AIcg)

**조건**

![웹훅연동조건](https://user-images.githubusercontent.com/48713266/164604297-25a3c37e-2ad0-4f8b-b549-c6a6f491a85b.png)

👉 적용

![웹훅연동조건적용](https://user-images.githubusercontent.com/48713266/164604301-8ac39909-1199-4da8-b74a-98e7ebfec33e.png)

💥 Json은 **JsonObject** 사용하도록 함

- 수정 전

  String을 그대로 파라미터로 사용했음


```
public void sendMessage(MessageForm msgForm) {
    String param = "{\"body\":\"" + msgForm.getBody() + "\",\n"
                 + "\"connectColor\":\"" + msgForm.getConnectColor() + "\",\n"
                 + "\"connectInfo\":[{\n"
                 + "                    \"title\":\"" + msgForm.getConnectInfo().getTitle() +"\",\n"
                 + "                    \"description\":\"" + msgForm.getConnectInfo().getDescription() + "\"\n"
                 + "                }]\n"
                 + "}";

    try {
        URL url = new URL(jandiUrl);
        HttpURLConnection con = (HttpURLConnection) url.openConnection();
        con.setRequestMethod("POST");
        con.setRequestProperty("Content-Type", "application/json");
        con.setRequestProperty("charset", "utf-8");
        con.setRequestProperty("Accept", "application/json");
        con.setDoOutput(true);

        try(OutputStream os = con.getOutputStream()) {
            byte[] input = param.getBytes("utf-8");
            os.write(input, 0, input.length);
        }

        try(BufferedReader br = new BufferedReader(new InputStreamReader(con.getInputStream(),"utf-8"))) {
            StringBuilder response = new StringBuilder();
            String responseLine = null;
            while ((responseLine = br.readLine()) != null) {
                response.append(responseLine.trim());
            }
        }
    } catch (IOException e) {
        logger.error(e.getMessage());
    } catch (ParseException e) {
        e.printStackTrace();
    }
}
```

- 수정 후

  lib폴더에 json-simple-1.1.1.jar 추가


```
public void sendMessage(MessageForm msgForm) {
    String param = "{\"body\":\"" + msgForm.getBody() + "\",\n"
                 + "\"connectColor\":\"" + msgForm.getConnectColor() + "\",\n"
                 + "\"connectInfo\":[{\n"
                 + "                    \"title\":\"" + msgForm.getConnectInfo().getTitle() +"\",\n"
                 + "                    \"description\":\"" + msgForm.getConnectInfo().getDescription() + "\"\n"
                 + "                }]\n"
                 + "}";

    try {				
				/*---추가한 Json Object 라이브러리 사용---*/
        JSONParser jsonParser = new JSONParser();
        Object obj = jsonParser.parse(param);
        JSONObject jsonObj = (JSONObject) obj;
				/*--------------------------------------*/

        URL url = new URL(jandiUrl);
        HttpURLConnection con = (HttpURLConnection) url.openConnection();
        con.setRequestMethod("POST");
        con.setRequestProperty("Content-Type", "application/json");
        con.setRequestProperty("charset", "utf-8");
        con.setRequestProperty("Accept", "application/vnd.tosslab.jandi-v2+json");
        con.setDoOutput(true);

        try(BufferedWriter bw = new BufferedWriter(new OutputStreamWriter(con.getOutputStream()))) {
            bw.write(jsonObj.toJSONString());
            bw.flush();
        }

        try(BufferedReader br = new BufferedReader(new InputStreamReader(con.getInputStream(),"utf-8"))) {
            StringBuilder response = new StringBuilder();
            String responseLine = null;
            while ((responseLine = br.readLine()) != null) {
                response.append(responseLine.trim());
            }
        }
    } catch (IOException e) {
        logger.error(e);
    } catch (ParseException e) {
        logger.error(e);
    }
}
```

- 알람전송 플로우차트

![6주차플로우차트](https://user-images.githubusercontent.com/48713266/164604322-1ce8301c-7e93-4c02-8e8b-3bd423f3b557.png)

- 알람전송 확인 화면

![알람](https://user-images.githubusercontent.com/48713266/164604332-78e39b76-4ef5-4a3f-a411-5012c125456b.png)
