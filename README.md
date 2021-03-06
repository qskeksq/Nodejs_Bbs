# Nodejs Bbs Server 
- 1. Server에서 서버 생성
- 2. 라우터에서 URL 분석 -> pathname, method 분석
- 3. 컨트롤러가 모든 내부 로직 처리
- 4. dao에 쿼리, 콜백 보내주기

# Server

## (1) Server 모듈

```javaScript
var router = require('./a_router');

var server = http.createServer((request, response)=>{
    // 경로 라우팅
    router.process(request, response);
});
```

## (2) Router.js 모듈
- url, querystring 모듈 사용 요청 url 분리 
- path 분기
- method 분기

```javaScript
exports.process = function(request, response){
    // url 분리
    var url = u.parse(request.url);
    var cmds = url.pathname.split('/');
    var method = request.method.toLowerCase();
    // 1. path 처리
    if(cmds[1] == 'bbs'){
        // 2. method 처리
        if(method == 'get'){
            var query = qs.parse(url.query);
            bbs.read(request, response, query);
        } else {
            // GET을 제외한 나머지 처리 방식이 비슷함을 알 수 있다
            var body = "";
            request.on('data', (data)=>{
                body += data;
            });
            request.on('end', ()=>{
                var bbsObj = JSON.parse(body);
                if(method == 'post'){
                    bbs.create(request, response, bbsObj);
                } else if(method == 'put'){
                    bbs.update(request, response, bbsObj);
                } else if(mtehod == 'delete'){
                    bbs.remove(request, response, bbsObj);
                }
            });
        }
    }
}
```

## (3) Controller 모듈
- 내부 로직 처리
- 쿼리 만들어 dao에 인자로 넘겨줌
- dao의 콜백 메소드 구현 장소

### create

```javaScript
exports.create = function(request, response, bbsObj){
    // 1. insert할 bbs 객체 넘겨줌 2. 콜백 메소드 구현
    dao.create(bbsObj, (result_code)=>{
        var result = {
            code : result_code,
            msg : '입력완료'
        }
        // 넘겨줄 때 객체 자체로 넘겨주면 First Argument should be... 오류 뜸.
        // 반드시 Json string으로 변형해서 넘겨준다
        response.end(JSON.stringify(result));
    });
}
```

### read

```javaScript
exports.read = function(request, response, search){
    // 넘겨줄 쿼리(read는 어떤 것을 읽을지 넘겨줘야 한다)
    var query = {};
    if(search.type === 'all'){
        query = {page:parseInt(search.page)};
    } else if(search.type === 'no'){
        query = {_id : -1};
		query._is = ObjectId(search._id)
    }
    // 1. dao에 넘겨줌 2. 넘어온 dataSet을 result에 넣어서 json으로 바꾼 뒤 넘겨준다
    dao.read(query, (dataSet)=>{
        console.log(dataSet);
        var result = {
            code : 200,
            msg : "정상처리",
            data : dataSet
        };
        response.writeHead(200, {'Content-Type':'application/json; charset=utf8'})
        response.end(JSON.stringify(result));
    });
}
```

## (4) Dao 모듈
- 데이터베이스(mongo, mysql)에 connect/createConnect/커넥션 풀 생성
- 넘어온 데이터베이스에 쿼리(insert(), find(), update(), remove());

### Create

```javaScript
exports.create = function(bbs, callback){
    mongo.connect(dbUrl, (err, db)=>{
        db.collection(table).insert(bbs); // NoSql이라 insert가 실패할 수가 없다
                                          // 자동으로 database와 table이 생성된다
        if(err){
            callback(400);
        } else {
            callback(200);
        }
        db.close();
    });
}
```

### Read

```javaScript
exports.read = function(search, callback){
    mongo.connect(dbUrl, (err, db)=>{
        // 쿼리
        // var projection = {title:'1'};
        // var projection = {_id:1} // 1이면 가져오고 0이면 가져오지 않는다.
        // 소팅
        var s = {
            _id : -1 // 1:내림차순, -1:오름차순
        }
        // 시작점
        // skip = 카운트를 시작할 index의 위치
        // 가져올 개수
        // limit
        var start = (search.page -1)*pageCount; 
        // 사용하지 않는 검색 컬럼 삭제
        delete search.page;
        var cursor = db.collection(table).find(search).sort(s).skip(start).limit(pageCount);    
        cursor.toArray((err, documents)=>{
            if(err){
                callback(documents, err);
                console.log(err);
            } else {
                callback(documents, err);
            }
        });
        db.close();
    });
}
```

### test 데이터 10000

```javaScript
var users = ["aaa", "bbb", "ccc", "ddd", "eee", "fff", "ggg", "hhh", "iii", "jjj"]

mongo.connect(dbUrl, (err, db)=>{
    // 100번씩 100개 번 돌면서 1000개의 데이터를 저장한다
    for(var index=0; index<100; index++){
        var array = [];
        for(j=0; j<100; j++){
            var bbs = {
                title : "", // 랜덤한 텍스트를 조합해서 입력
                content : "내용입니다",
                user_id : users[10%j],   // 10개를 미리 정해놓고 random 입력
                date : new Date()+""
            }
            array[j] = bbs;
        }
        db.collection(table).insert(array, function(err, inserted){
            if(err){
                console.log(err);
            } else {
                console.log('성공');
            }
        });
    }
});
```

# Client

```java
/**
 * 1. GET
 * localhost:8090/bbs?type=all
 * localhost:8090/bbs?type=all&no=1
 * 2. POST
 * localhost:8090/bbs?type=all
 */
public static String BASE_URL = "http://192.168.0.140:8090/bbs/";
public static String TYPE_ALL = "?type=all";
public static String TYPE_NO = "?type=no";
public static String QUERY_NO = "&no=";
```

### GET AyncTask

```java
public static void loadGet(final TaskInterface taskInterface, final String seq) {

    new AsyncTask<String, Void, String>() {
        @Override
        protected String doInBackground(String... seqs) {
            String result = "";
            if (seqs[0] == null || seqs[0].equals("")) {
                // http://192.168.0.140:8090/bbs/?type=all
                result = BbsRemote.sendGet(BASE_URL + TYPE_ALL);
                Log.e("url 확인", BASE_URL + TYPE_ALL);
            } else {
                // http://192.168.0.140:8090/bbs/?type=all&no=1
                result = BbsRemote.sendGet(BASE_URL + TYPE_NO + QUERY_NO + seqs[0]);
                Log.e("url 확인", BASE_URL + TYPE_NO + QUERY_NO + seqs[0]);
            }
            Log.e("결과 확인", result);
            return result;
        }

        @Override
        protected void onPostExecute(String jsonString) {
            Result datas = new Gson().fromJson(jsonString, Result.class);
            taskInterface.setData(datas.getData());
        }
    }.execute(seq);
}
```

### POST AysncTask

```java
public static void loadPost(final TaskInterface taskInterface, final Data data) {

    new AsyncTask<Data, Void, String>() {
        @Override
        protected String doInBackground(Data... json) {
            Gson gson = new Gson();
            String result = BbsRemote.sendPost(BASE_URL, gson.toJson(data));
            return result;
        }

        @Override
        protected void onPostExecute(String jsonString) {
            Result data = new Gson().fromJson(jsonString, Result.class);
            taskInterface.setResult(data);
        }
    }.execute(data);
}
```

### POST 스트림

```java
public static String sendPost(String address, String postData) {
    String result = "";
    try {
        URL url = new URL(address);
        HttpURLConnection con = (HttpURLConnection) url.openConnection();
        con.setRequestMethod("POST");
        // Header 작성
        con.setRequestProperty("Content-Type", "application/json; charset=utf8");
        con.setRequestProperty("Authorization", "token=나중에 서버랑 통신할 때 쓸거임");

        con.setDoOutput(true);
        OutputStream os = con.getOutputStream();
        os.write(postData.getBytes());
        os.flush();
        os.close();

        // 통신이 성공인지 체크
        if (con.getResponseCode() == HttpURLConnection.HTTP_OK) {
            // 여기서 부터는 파일에서 데이터를 가져오는 것과 동일
            InputStreamReader isr = new InputStreamReader(con.getInputStream());
            BufferedReader br = new BufferedReader(isr);
            String temp = "";
            while ((temp = br.readLine()) != null) {
                result += temp;
            }
            br.close();
            isr.close();
        } else {
            Log.e("ServerError", con.getResponseCode() + "");
        }
        con.disconnect();
    } catch (Exception e) {
        Log.e("Error", e.toString());
    }
    Log.e("결과", result);
    return result;
}
```
### GET 스트림

```java
public static String sendGet(String address) {
    String result = "";
    try {
        URL url = new URL(address);
        HttpURLConnection con = (HttpURLConnection) url.openConnection();
        con.setRequestMethod("GET");

        // 통신이 성공인지 체크
        if (con.getResponseCode() == HttpURLConnection.HTTP_OK) {
            // 여기서 부터는 파일에서 데이터를 가져오는 것과 동일
            InputStreamReader isr = new InputStreamReader(con.getInputStream());
            BufferedReader br = new BufferedReader(isr);
            String temp = "";
            while ((temp = br.readLine()) != null) {
                result += temp;
            }
            br.close();
            isr.close();
        } else {
            Log.e("ServerError", con.getResponseCode() + "");
        }
        con.disconnect();
    } catch (Exception e) {
        Log.e("Error", e.toString());
    }
    Log.e("결과", result);
    return result;
}
```

### 페이지 제한하기

- RecyclerView의 addOnScrollListener, LayoutManager의 findFirstVisibleItemPosition()사용

```java
recyclerview.addOnScrollListener(new RecyclerView.OnScrollListener() {
    @Override
    public void onScrollStateChanged(RecyclerView recyclerView, int newState) {
        super.onScrollStateChanged(recyclerView, newState);
        // 멈춤 상태가 되면 데이터를 불러온다
        if(newState == RecyclerView.SCROLL_STATE_IDLE && isLastItem){
            BbsRemote.loadGet(MainActivity.this, null);
        }
    }

    @Override
    public void onScrolled(RecyclerView recyclerView, int dx, int dy) {
        super.onScrolled(recyclerView, dx, dy);
        // 리스트뷰와 다르게 레이아웃 매니저에서 꺼내 사용한다
        int firstVisiblePosition = manager.findFirstVisibleItemPosition();
        int lastVisiblePosition = manager.findLastVisibleItemPosition();
        int visibleCount = manager.getChildCount();
        int total = manager.getItemCount();

        if(lastVisiblePosition == total-1){
            isLastItem = true;
        } else {
            isLastItem = false;
        }
    }
});
```

```java
public void setData(Data[] newData) {
    if (datas.length == 0) {
        this.datas = newData;
    } else {
        Data[] temp = new Data[datas.length + newData.length];
        // 1. 이놈을 2.여기서부터 3.temp배열에 4.temp의 0부터 5.datas의 길이만큼 복사한다
        // 먼저 기존의 배열 복사
        System.arraycopy(datas, 0, temp, 0, datas.length);
        // 나중에 들어온 배열 복사
        System.arraycopy(newData, 0, temp, datas.length, newData.length);
        datas = temp;
    }
       notifyDataSetChanged();
    Log.e("크기", datas.length+"");
}
```


# 데이터베이스

### 서버에 파일 전송

```javaScript
var server = http.createServer((request, response)=>{
    if(request.url == "/upload"){
        var form = new formidable.IncomingForm();
        // 파일 2개 이상
        form.multiples = true;
        form.on('fileBegin', function (name, file){
             // 파일이 들어오기 시작할 때
             // 내가 원하는 경로에 파일이 저장되도로 하기 위해 경로 지정
             file.path = './public/img/' + file.name;
        });

        form.parse(request, (err, fieldNames, files)=>{ // 임시폴더에 저장
            console.log(files);
            console.log(fieldNames);
            if(err){
                console.log(err);
            } else {
                // 멀티플로 설정한 경우는 배열에서 꺼내서 해야 한다
                for(i in files){
                    var oldpath = files[i].path;
                    var realpath = "c:/temp/upload/"+files[i].name;
                    fs.renameSync(oldpath, realpath);
                }
                response.end('complete');
            }
        })
    } else {
        response.end('404 not found');
    }
});
```

### MySql 데이터베이스 연결

```java
var mysql = require('mysql');

var settings = {
    host : "localhost",
    user  : 'root',
    password : 'mysql',
    port:'3306',
    database : 'memo'
}

var con = mysql.createConnection(settings);


con.connect();
con.query('select * from memo', (error, record_set, columns)=>{
    record_set.forEach(function(record) {
        console.log(record);
    });
    // this.end();     // 쿼리 처리에 대한 연결 해제
});

con.end();  // 데이터베이스 연결 해제
```
