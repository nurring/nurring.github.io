---
title:  "디바이스 리포트 & 실시간 모니터링 웹 프로그램"
period:   2019.10.21 - 2019.11.28
category: portfolios
tags: 
   - Spring
   - Mybatis
toc: true
toc_sticky: true
---

## 개발 환경
+ Java
+ Spring4
+ myBatis
+ Mysql
+ Tomcat9
<br>


## 개발 개요
IoT 기기의 데이터를 실시간으로 모니터링하고 통계와 함께 리포트하는 웹 프로그램을 구현하였다.   
통계 그래프 구현을 위한 컬럼과 테이블 조인을 세부적으로 조정하기 위하여 myBatis를 이용하였으며, 실시간 데이터 감지 및 그래프 업데이트를 위해 ajax polling을 적용하였다.



## DB 설계
![erd](/assets/images/erd_devicemntr.png)

<br>


## 주요 사용 기능
Maven 프로젝트에 의존성을 주입하였다. myBatis *Mapper.xml 파일에 쿼리를 작성하고 도메인 파라미터와 매핑하였으며, ajax 통신을 위하여 Jackson을 이용해 json 형식의 데이터를 리턴하도록 하였다.

### 환경 설정
pom.xml에 다음과 같이 mybatis, Jackson 의존성을 주입하였다.   

```xml
//pom.xml
<!-- myBatis -->
<dependency>
   <groupId>org.mybatis</groupId>
   <artifactId>mybatis</artifactId>
   <version>3.4.1</version>
</dependency>
<dependency>
   <groupId>org.mybatis</groupId>
   <artifactId>mybatis-spring</artifactId>
   <version>1.3.0</version>
</dependency>
<!-- Jackson -->
<dependency>
   <groupId>com.fasterxml.jackson.core</groupId>
   <artifactId>jackson-core</artifactId>
   <version>2.10.1</version>
</dependency>
<!-- Jackson-databind -->
<dependency>
   <groupId>com.fasterxml.jackson.core</groupId>
   <artifactId>jackson-databind</artifactId>
   <version>2.10.1</version>
</dependency>
```
<br>
root-context 파일에 jdbc, myBatis 설정파일 및 매퍼, 세션 정보를 bean 등록했다.
```xml
//root-context.xml
<bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
   <property name="driverClassName" value="com.mysql.cj.jdbc.Driver"></property>
   <property name="url" value="jdbc:mysql://ip주소:3306/DB명?useSSL=false&amp;serverTimezone=UTC"></property>
   <property name="username" value="유저 아이디"></property>
   <property name="password" value="비밀번호"></property>
</bean>	

<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
   <property name="dataSource" ref="dataSource"></property>
   <property name="configLocation" value="classpath:/mybatis-config.xml"></property>
      //myBatis 관련 설정을 가져옴	
   <property name="mapperLocations" value="classpath:mappers/**/*Mapper.xml"></property>
      //mappers 패키지 하위에 포함 된 파일명이 Mapper로 끝나는 모든 xml 파일을 mapper로 가져옴
</bean>

<bean id="sqlSession" class="org.mybatis.spring.SqlSessionTemplate" destroy-method="clearCache">
   <constructor-arg name="sqlSessionFactory" ref="sqlSessionFactory"></constructor-arg>
</bean>
```
<br>

### *Mapper.xml ~ 테이블 간 일대다 연결
도메인과 매핑 될 컬럼, 테이블 정보를 명시하고 데이터를 조회해 올 쿼리를 직접 작성했다.   
테이블 간 관계의 경우, 자식 테이블에서 가져올 컬럼들을 resultMap에 collection으로 명시하였다. 그리고 Join 쿼리에서 resultMap을 명시하여 테이블을 매핑시켰다.   
Left Join을 이용하여 자식 테이블을 기준으로 부모 테이블의 데이터가 추가될 수 있게 Join 쿼리를 작성하였으며,   
실제 데이터 형식은 resultMap 정보를 통해, 쿼리에 명시된 부모 테이블 컬럼에 collection으로 자식 테이블 컬럼들이 따라오게 된다. 

```xml
//errorMapper.xml
<mapper namespace="ErrorMapper">	
   <resultMap id="DeviceResult" type="DeviceVO">
      <id property="device_id" column="device_id" />
      <result property="device_name" column="device_name"/>
      <result property="device_address" column="device_address"/>
      <result property="device_latitude" column="device_latitude"/>
      <result property="device_longitude" column="device_longitude"/>
      <result property="regidate" column="regidate"/>
      <result property="updatedate" column="updatedate"/>
      <collection property="errorList" ofType="ErrorVO" resultMap="ErrorResult"/>
   </resultMap>
   
   <resultMap id="ErrorResult" type="ErrorVO">
      <id column="e_id" property="e_id"/>
      <result column="device_ip" property="device_ip"/>
      <result column="err_code" property="err_code"/>
      <result column="err_type" property="err_type"/>
      <result column="err_message" property="err_message"/>
      <result column="server_time" property="server_time"/>
      <result column="count" property="count"/>
      <result column="month" property="month"/>
      <result column="year" property="year"/>
   </resultMap>
...
   <select id="err24" resultMap="DeviceResult">
   SELECT 
      a.device_id, a.device_name, a.device_ip, a.device_latitude, a.device_longitude, b.err_code, b.err_type, b.err_message, b.server_time
   FROM
      (SELECT *
      FROM gztls_error
      WHERE server_time BETWEEN #{from} and #{til} ) b
   LEFT JOIN gztls_device a
   ON a.device_ip = b.device_ip
   ORDER BY b.server_time DESC;
   </select>
...
```
<br>

### *Mapper.xml ~ 동적 SQL
myBatis가 제공하는 Dynamic SQL 기능으로 쿼리의 재활용성을 높였다._[myBatis Reference 참조](https://mybatis.org/mybatis-3/dynamic-sql.html)_   
아래와 같이 조건에 따라 쿼리가 변형 적용되기 때문에 효율적으로 원하는 데이터를 가져올 수 있다.

```xml
//errorMapper.xml
...
<select id = "selectErrCnt" resultMap="ErrorResult">
SELECT err_type, COUNT(err_type) as count
FROM gztls_error
<where>
   <if test='server_time != null'>
   server_time BETWEEN #{server_time} AND #{to_date}
   </if>
</where> 
GROUP BY err_type;	
</select>
...
```
<br>
복잡한 조건 등의 이유로 파라미터를 여러개 받아오는 일이 잦아 map 방식으로 파라미터를 받아와 명확히 구분되도록 하였다.
```java
//ErrorDAOImpl.java
...
@Override
public List<ErrorVO> selectErrCnt(Map<String, String> map) {
   return sqlSession.selectList(SelectErrCnt, map);
}
...
```
```java
//ErrorServiceImpl.java
...
@Override
public List<ErrorVO> selectErrCnt(Map<String, String> map) {
   return dao.selectErrCnt(map);
}
...
```
```java
//RController.java
...
@RequestMapping(value="/errcntjsn", method=RequestMethod.GET)
public List<ErrorVO> errcnt(@RequestParam Map<String, String> param) {	
   logger.info("param..........."+param);
   List<ErrorVO> vos = es.selectErrCnt(param);
   return vos;		
}
...
```
<br>

### @ResponseBody와 Jackson을 이용한 json 리턴

```java

```
<br>

### 2차 캐시 적용
Configuration 자바파일에 캐시가 적용될 위치를 명시한다. Ignite 설정 부분만 따로 뽑아 xml로 만들어 Configuration 파일에 Import했다.   
어노테이션에 정의한 region을 value값으로 준다. 자식 엔티티로 연결된 객체도 추가로 명시해야 한다.

```xml
//applicationContext-ignite.xml
<bean parent="transactional-cache">
   <property name="name" value="UcUser" />
</bean>
<bean parent="transactional-cache">
   <property name="name" value="UcPhone" />
</bean>
<bean parent="transactional-cache">
   <property name="name"
      value="UcUser.phones" />
</bean>
```

```java
//SpringConfiguration.java
@Configuration
@ImportResource(locations = "classpath:applicationContext-ignite.xml") //ignite 설정
......
public class SpringConfiguration {
......
   public Properties additionalProperties() {
   ......
      properties.setProperty(AvailableSettings.USE_SECOND_LEVEL_CACHE, Boolean.TRUE.toString()); //2차 캐시 사용
      properties.setProperty(AvailableSettings.USE_QUERY_CACHE, Boolean.TRUE.toString()); //쿼리 캐시 사용
      properties.setProperty(AvailableSettings.GENERATE_STATISTICS, Boolean.TRUE.toString()); //캐시 결과 로그 출력
      properties.setProperty(AvailableSettings.CACHE_REGION_FACTORY, HibernateRegionFactory.class.getName()); //캐시 대상

      properties.setProperty("org.apache.ignite.hibernate.ignite_instance_name", "cafe-grid");
      properties.setProperty("org.apache.ignite.hibernate.default_access_type", "NONSTRICT_READ_WRITE");
```
<br>
엔티티 레벨의 캐시가 가능하도록 클래스에 캐시 어노테이션을 명시한다.   
관리 모듈이므로 분산 간 트랜잭션 동시성을 예민하게 고려하지 않고 READ_WRITE/NONSTRICT_READ_WRITE 설정을 사용하였다.   
두 엔티티와 연결된 객체에 설정하였다.


```java
//UcUser.java
@Cachable
@Cache(region="UcUser", usage = CacheConcurrencyStrategy.READ_WRITE)
@Entity
public class UcUser implements Serializable {
......
	@Cache(region="UcUser.phones", usage = CacheConcurrencyStrategy.NONSTRICT_READ_WRITE)
	@OneToMany(fetch=FetchType.LAZY, cascade=CascadeType.ALL, mappedBy="ucUser")
	private List<UcPhone> phones;
```

```java
//UcPhone.java
@Cacheable
@Cache(region="UcPhone", usage = CacheConcurrencyStrategy.READ_WRITE)
@Entity
public class UcPhone implements Serializable {
```
<br>


## 구현 기능
### 사용자 조회

컬럼 조건에 따라 사용자 검색이 가능하도록 구현하였다. 

```java
//UcController.java
@PostMapping(value = "/search.html")
public String search(String type, String search, Model model) {
   logger.info("search started targeting ->" + search + "===================================");
   model.addAttribute("search", search);
   if (type.contentEquals("byall")) { // 전체 검색
      
      List<UcUser> result = new ArrayList<UcUser>(); //최종 결과물 들어갈 리스트
      List<UcUser> ucuserAll = ucService.findAll(); //전체 유저 정보 몽땅 가져와서			
      for (UcUser usertmp : ucuserAll) { //유저마다 살펴보는데
         if(usertmp.getName().toLowerCase().contains(search.toLowerCase())) { //대소문자 구분 없이 검색
            result.add(usertmp);
         } else if(usertmp.getAddress().toLowerCase().contains(search.toLowerCase())) {
            result.add(usertmp);
         } else if(usertmp.getBusinessName().toLowerCase().contains(search.toLowerCase())) {
            result.add(usertmp);
         } else if(usertmp.getDepartmentName().toLowerCase().contains(search.toLowerCase())) {
            result.add(usertmp);
         } else {
            List<UcPhone> ucphone = usertmp.getPhones(); //유저가 가지고있는 연락처 정보를 리스트에 넣고					
            for (UcPhone phonetmp : ucphone) { //그 리스트 보기						
               if(phonetmp.getPhone().contains(search)) {
                  result.add(usertmp);
                  break;
               }						
            }
         }
      }
      model.addAttribute("searches", result);
   } else if (type.contentEquals("byname")) { //이름 포함
      model.addAttribute("searches", ucService.findAllByNameContaining(search));
   } else if (type.contentEquals("bybusiness")) { //회사명 포함
      model.addAttribute("searches", ucService.findAllByBusinessNameContaining(search));
   } else if (type.contentEquals("bydepartment")) { //부서명 포함
      model.addAttribute("searches", ucService.findAllByDepartmentNameContaining(search));
   } else if (type.contentEquals("byaddress")) { //주소 포함
      model.addAttribute("searches", ucService.findAllByAddressContaining(search));
   } else if (type.contentEquals("byphone")) { //연락처 포함
      List<UcUser> ucuser = new ArrayList<UcUser>();
      List<UcPhone> phones = ucService.findAllUcPhoneByPhoneContaining(search);
      for (UcPhone ucPhone : phones) { 
         ucuser.add(ucPhone.getUcUser());
      }
      model.addAttribute("searches", ucuser);
   }
   return "search";
}
```
<br>
FetchType을 LAZY로 설정하였기 때문에 부모 엔티티를 조회하면 자식 객체와 연동된 리스트에 실제 데이터는 없고 프록시만 존재한다. 
그러므로 세션이 닫힐 경우 부모와 자식 객체는 분리되어 있기 때문에 더 이상 데이터를 호출할 수 없게 된다.   
위의 이유로 자식 데이터까지 select 하는 메소드가 실행되고 세션이 닫힐 경우, 초기화 되지 않은 프록시에 의해 LazyInitializationException이 발생하였다.   
그래서 자식 데이터를 get해야하는 메서드의 경우 자식 객체를 **Hibernate.initialize()**하여 프록시 강제 초기화 시켰다.


```java
//UcService.java
public List<UcUser> findAllByNameContaining(String name){
   List<UcUser> ucuser = userRepository.findAllByNameContaining(name);
   for (UcUser tmp : ucuser) {
      Hibernate.initialize(tmp.getPhones()); //lazy
   }
   return userRepository.findAllByNameContaining(name);
}
```
<br>

### 페이지네이션
JPA가 제공하는 Pageable을 이용하여 페이지네이션을 구현하였다.   
page변수를 받으면 View단에 뿌려줄 버튼의 갯수와 숫자를 함수로 구해 model에 담아서 전달하고 View단에서는 반복문으로 버튼을 생성하였다.


```java
//UcController.java
@GetMapping(value = "/{page}")
	public String list(@PathVariable("page") Integer page, Model model) {
		logger.info("list.jsp started!===================================");
		Pageable pageable = PageRequest.of(page, 10);				
		
		if (page<0 || page==null) { //오류 처리
			page = 0;
		}		
		//뽑혀야 할 전체 페이지버튼 수
		long cntTmp = ucService.userCount();
		double cntTmp2 = cntTmp/10.0;
		long finalVal = (long) (Math.ceil(cntTmp2));
		
		//한 화면에 보여줄 첫번째 버튼과 마지막 버튼(ex.1과5, 6과 10..)
		long startTmp = page/5;
		long startRange = startTmp*5;
		long endRange =((startTmp+1)*5)-1;
		if (endRange > finalVal) { //맨 마지막 화면에서는 다섯개 버튼X, 마지막 숫자에서 끊어주는 작업
			endRange = finalVal-1;
		}
		if (endRange < 0) { //데이터가 0개일 경우
			endRange = 0;
		}		
		model.addAttribute("page", page);
		model.addAttribute("totalCount", cntTmp);
		model.addAttribute("startRange", startRange);
		model.addAttribute("endRange", endRange);
		model.addAttribute("totalPage", finalVal);
		model.addAttribute("users", ucService.findAllPage(pageable));
		return "list";
	}
```

``` jsp
//list.jsp
<div class="paging">
   <div class="ui animated button" tabindex="0" onclick="location.href='/0'"> <!-- 맨 앞으로 -->
      <div class="visible content">《</div>
      <div class="hidden content">Head</div>
   </div>
   <c:if test="${!users.first}"> <!-- 이전으로 -->
      <div class="ui animated button" tabindex="0" onclick="location.href='${users.number-1}'"> 
         <div class="visible content">〈</div>
         <div class="hidden content">Prev</div>
      </div>
   </c:if>

   <c:forEach begin="${startRange }" end="${endRange }" var="e"><!--페이지 버튼 출력-->
      <button type="button" class="ui button" id="button${e }" onclick="location.href='${e }'">${e+1 }</button>
   </c:forEach>

   <c:if test="${!users.last}"><!--다음으로-->
      <div class="ui animated button" tabindex="0" onclick="location.href='${users.number+1}'">
         <div class="visible content">〉</div>
         <div class="hidden content">Next</div>
      </div>
   </c:if>
   <div class="ui animated button" tabindex="0" onclick="location.href='/${totalPage-1 }'"> <!--맨 뒤로-->
      <div class="visible content">》</div>
      <div class="hidden content">Tail</div>
   </div>
</div>
```
<br>

### 음성 검색
Chrome의 음성 인식 API를 이용하여 JavaScript로 처리하였다.

```javascript
//list.jsp
if ('SpeechRecognition' in window) {
    console.log("음성인식을 지원하는 브라우저입니다.")
 }
 try {
    var recognition = new(window.SpeechRecognition || window.webkitSpeechRecognition
    		|| window.mozSpeechRecognition || window.msSpeechRecognition)();
 } catch (e) {
    console.error(e);
 }
 
 var getvoices = false;
 
function check_speech(){  
    recognition.lang = 'ko-KR';
    recognition.interimResults = false;
    recognition.maxAlternatives = 1;
    recognition.continuous = true;
    if(!getvoices) speech_to_text();
    else stop();
 }
 function speech_to_text() {
    getvoices = true;

    recognition.start();
    var resText = "";

    recognition.onstart = function() {
       console.log("음성인식이 시작 되었습니다.")
       document.getElementById("speech").value = "인식중..";
    }
    recognition.onresult = function(event) {
       console.log('You said: ', event.results[0][0].transcript);
       resText = resText + event.results[0][0].transcript;
       document.getElementById("search").value = resText;
    };
    recognition.onend = function() {
       document.getElementById("speech").value = "음성인식";
    }
 }
 function stop() {
    recognition.stop();
    document.getElementById("speech").value = "음성인식";
    getvoices = false;
 }
```
<br>

### 사용자 CRUD - 수정 날짜 기록


정보 수정 시 마지막 수정일이 업데이트 되도록 구현하였다.    
insert와 update가 한 메서드에서 이루어지므로 update시 등록일자까지 동시에 변경되지 않도록 등록일 변수는 updatable을 false로 지정하였다.   
이렇게 설정하면 Controller에서 서버시간을 set하여도 쿼리 자체에 해당 컬럼이 포함되지 않는다.

```java
//UcUser.java
@Column(columnDefinition="TIMESTAMP DEFAULT CURRENT_TIMESTAMP", updatable = false, insertable = true)
	private Date registeredOn; //등록일
	
@Column(columnDefinition="TIMESTAMP DEFAULT CURRENT_TIMESTAMP", updatable = true, insertable = true)
private Date lastUpdatedOn; //마지막 수정일
```

```java
//UcController.java
@PostMapping(value = "/save.html")
@DateTimeFormat(pattern = "yyyy-MM-dd")
public String save(UcUser ucUser) {
   logger.info("save started targeting ->" + ucUser.getName() + "===================================");
   try {
      Calendar calt = Calendar.getInstance();
      ucUser.setLastUpdatedOn(calt.getTime());
      ucUser.setRegisteredOn(calt.getTime());
      ucService.save(ucUser);
   } catch (Exception e) {
      e.printStackTrace();
   }
   return "redirect:/0";
}
```
<br>

### 연락처 CRUD - 중복 검사
연락처 중복 검사를 Controller에서 구현할 경우 경고창 띄우기가 복잡하여 View단에서 ajax 비동기방식으로 처리하였다.

```javascript
//editform.jsp
function checking(){
	if ($('#phone').val()==""){
        alert('번호를 입력해 주세요');
        return false;
	}
	var userid = document.getElementById("userid").value;
	var phone = document.getElementById("phone").value;
	var carrier = document.getElementById("carrier").value;
	var set = {'phone': phone, 'carrier': carrier, 'userid': userid};
	
	$.ajax({
		url: "/save2.html",
		type: "POST",
		data: set,
		success: function(){
			alert("번호 등록 완료");
			location.replace("/oneview.html/"+ userid);
		},
		error: function(){
			alert("번호 중복");
		}	
	});
}
```

## 시연 영상 

 {% youtube "https://www.youtube.com/watch?v=xw-3VIGQBrM&t=1s" %}

## 프로젝트 회고
Spring Framework를 이용하여 MVC 모델에 확실히 적응하였다.   
초반 환경 설정이 가장 까다롭고 어려운 작업임을 느꼈고, Hibernate 다루기가 어색하였지만   
결과적으로 JDBC만을 이용하던 다른 프로젝트보다 쿼리 자체에 대한 의존도를 낮추고   
모듈화를 심화할 수 있었다.    
또한 1, 2차 캐시를 적용하여 이론으로 알고 있던 캐시를 직접 확인할 수 있었다.   