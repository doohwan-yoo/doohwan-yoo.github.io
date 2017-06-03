---
layout: post
title: "Springboot 에서 Elasticsearch 5.X 사용하기"
description: "Elasticsearch 5.X 가 현재 spring-data-elasticsearch 를 지원하지 않아 elasticsearch 에서 직접 제공하는 라이브러리를 사용한다."
tags: [springboot, elasticsearch5.x, org.elasticsearch]
---
#### 시작
회사에서 새 프로젝트를 하며, Elasticsearch 를 5.X 버전으로 새로 설치를 했다. 그리고 Springboot로 연동을 하려고 하는 도중 기존에 제공하는 spring-data-elasticsearch 와 호환성 문제로 제대로 동작하지 않는 것을 발견하고 5.x대 버전에 spring-data-elasticsearch를 열심히 찾아 보았다.
없었다.. 문서를 또 검색하고 검색했다. Spring 진영에서 답글이 오고 간것이 보였다. 사용자들이 원성이 자자했다 "5.x버전이 나온지 1년이 지났는데 왜 지원 안해주나요?", "현재까지 아무 소식 없음?". Spring 진영에서 답을 줬다... "우리 이걸 할 사람과 여력이 없어.". 그래서 깔끔하게 포기하고 Elasticsearch 에서 제공해주는 JDBC 라이브러리로 직접 간단한 예제를 만들어 봤다.

해당 소스는 아래 링크에서 문서를 참조하여 만들었다. 부족한 점이 있다면 아래 링크에서 원하는 부분을 찾아서 추가 구현 하시도록!
[https://www.elastic.co/guide/en/elasticsearch/client/java-api/current/index.html](https://www.elastic.co/guide/en/elasticsearch/client/java-api/current/index.html) 

### 준비하기
먼저 시작하기에 앞서서 음악을 검색한다고 가정하고 Elasticsearch 에 간단하게 인덱스를 생성하고 데이타를 연동했다.

```json
PUT /meta
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  },
  "mappings": {
    "song": {
      "properties": {
        "name": {"type": "text"}
      }
    }
  }
}
```
그리고 요즘 핫한 노래 '시그널' 을 넣어두었다.

이제 위에 보이는 name 값으로 elasticsearch 를 연동하여 시그널을 검색해 보는 예제를 만들어 보겠다

### 라이브러리
```gradle
  compile('org.elasticsearch:elasticsearch:5.1.1')
	compile('org.elasticsearch.client:transport:5.1.1')
	compile('org.apache.logging.log4j:log4j-to-slf4j:2.7')
```

위에 세개의 라이브러리가 필요하다. 버전은 편하신대로 맞추시라.
마지막에 log4j는 elasticsearch 가 자체 로깅을 하기 위해서 필요하다.

### YML 설정

```yaml
elasticsearch:
   cluster-name: music-application
   host: [IP]
   port : [PORT]
```
cluster 이름과 IP, PORT 를 넣어준다

### 설정파일 만들기

ESconfig.java 라는 파일을 만들어서 설정을 해보도록 하겠다.

```java
@Configuration
public class EsConfig {

    @Value("${spring.elasticsearch.cluster-name}")
    private String clusterName;

    @Value("${spring.elasticsearch.host}")
    private String host;

    @Value("${spring.elasticsearch.port}")
    private int port;

    @Bean
    public Client client() throws Exception {
        Settings settings = Settings.builder()
                .put("cluster.name", clusterName).build();

        return new PreBuiltTransportClient(settings)
                .addTransportAddress(new InetSocketTransportAddress(InetAddress.getByName(host), port));

    }
}
```

어렵지 않다. yaml 에서 설정한 값들을 불러와서, 아래 client bean 함수 안에 있는 Elasticsearch 에서 제공해주는 함수들을 사용하여 본인이 구축한 elasticsearch 와 접근할 수 있도록 한다.

### Repository 생성

Elasticsearch 에서 제공해주는 query builder 로 데이터를 가져올 Repository 를 생성해 보겠다.

```java
@Repository
public class EsRepository {
    private Client client;

    @Autowired
    public EsRepository(Client client) {
        this.client = client;
    }

    public HashMap<String, Object> findSongWithPrefix(String prefix) {

        SearchResponse response = client.prepareSearch("meta")
                .setTypes("song")
                .setSearchType(SearchType.DFS_QUERY_THEN_FETCH)
                .setQuery(QueryBuilders.prefixQuery("name", prefix))
                .setFrom(0).setSize(20).setExplain(true)
                .get();

        SearchHits hits = response.getHits();

        HashMap<String ,Object> result = new HashMap<>();

        result.put("total", hits.getTotalHits());
        result.put("contentsList", hits.getHits());

        return result;
    }
}
```
앞에서 만든 client bean 을 DI 해주고, 앞 글자를 기준으로 음악을 가져오는 `findSongWithPrefix` 함수를 만들었다. Elasticsearch 가 Querydsl 을 지원하기 때문에 client가 제공해주는 prepareSearch 함수는 Querydsl 과 유사한 면이 많다. 기존에 Elasticsearch 에 Querydsl 에 익숙하다면 사용법은 금방 익힐 수 있을 것이라고 본다.

결과는 HashMap 으로 임시로 만들었는데, 원한다면 vo 객체를 만들어서 처리하면 될 것 같다.

### 결과 확인
```java
@Test
public void test1() {
	HashMap<String, Object> result = esRepository.findSongWithPrefix("시그");


	SearchHit [] hits = (SearchHit [])result.get("contentsList");
	System.out.println("Total : " + result.get("total"));
	System.out.println("Song Name : " + hits[0].getSource().get("name"));
}
```
Unit Test 를 진행하였고, 결과는?

```java
Total : 1
Song Name : 시그널
```

잘 나온다.

### 남은 과제
현재 서버와 Elasticsearch 사이에 커넥션은 하나만 연결 되는 것으로 되어있다. Connection pool 이 필요한 경우를 고려해 추가 구현이 필요해 보인다. 또한 원하는 질의에 대한 결과에 대해서 매번 수작업으로 데이터를 본인이 만든 데이터 객체로 옮겨야 하는데, Reflection 을 쓰던 해서 자동화할 수 있게 만들 미션들이 남아있다. 그 전에 5.x 대 버전을 지원해줘!

일단 위에 예제들은 급한 불을 끄기 위해 만든 상황이다. 기존에 것을 쓰고 싶으시다면 2.x대 버전으로 다운그레이드 하기를 추천 드립니다.

Thanks You!
