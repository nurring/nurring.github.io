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
ResposneBody 어노테이션으로 컨트롤러에서 데이터가 View가 아닌 Response Body로 리턴되게 한다. View로 리턴되는 함수와 구분하기 위하여 컨트롤러를 둘로 나누어 사용했다. 그리고 각각 함수마다 어노테이션을 달아주는 대신 클래스에 @RestController를 명시하였다. 복수의 파라미터는 map방식으로 받아온다.  
객체 리턴은 Jackson을 이용하여 json 형식으로 변환되도록 했다.

```java
//RController.java
@RestController
public class RController {
...
	@RequestMapping(value="/searchjsn", method=RequestMethod.GET)
	public List<DeviceVO> byname(@RequestParam Map<String, String> param) {
		logger.info("param..........."+param);
		List<DeviceVO> vo = ds.selectDevice(param);
		logger.info(vo.toString());
		return vo; //json형식으로 리턴
	}
...
```
<br>


## 구현 기능
### Chart.js를 이용한 통계 차트
데이터 입력을 실시간으로 감지하여 차트에 반영될 수 있게 구현하였다. setInterval로 주기적으로 ajax 통신으로 새로운 데이터를 확인하게 하였다.   
가져온 데이터는 Chart.js API 데이터셋 형식에 맞게 가공하여 업데이트하였다.   
Chart.js API를 이용하여 그래프를 출력하였다. 비동기 통신으로 가져온 데이터를 Chart.js 데이터셋 형식에 맞게 가공하여 업데이트 하였다.   
차트를 출력하는 데는 크게 label과 dataset이 필요하다. label은 가로축이 되고, dataset 리스트에 들어있는 각각이 또다시 data, label, color등 정보를 갖게 된다.   
차트 업데이트는 제공되는 update함수를 이용해야 하며, 이전에 그려진 차트가 있다면 destroy 함수로 차트를 지우고 그려야 마우스오버 시 그래프가 겹치지 않는다.

```javascript
//errbytime.jsp
...
$.ajax({
   url : "cntbymonthjsn",
   type : "GET",
   data : set,
   error : function() {
      console.log(error);
   }
}).done(function(results){
   console.log(results);
   dname=[];dtsize=[];dtsetArr=[]; //배열 초기화
   for(i in results){		
      dname.push(results[i].device_name);		
      dtset=[];lbl=[];	
      dtsize.push(results[i].errorList.length);		
      for(j in results[i].errorList){							
         dtset.push(results[i].errorList[j].count);	
         lbl.push(results[i].errorList[j].month+"월");		
      }
      dtsetArr.push(dtset);			
   }		
   lineChartData = { labels: lbl, datasets: [] }, //큰 데이터셋 구조를 만들고
   array = dtsetArr;
   label = dname; //정제한 데이터를 가져온 다음

   array.forEach(function (a, i) {
      lineChartData.datasets.push({ //데이터셋에 반복문으로 넣는다
         backgroundColor: colors[i],
         data: a,
         label: dname[i]
      });
   });		
   console.log("lineChartData",lineChartData);
   updateYear();
...
function updateYear(){
   myChart.data = lineChartData;
	myChart.update();
};
...
```
<br>

### Kakao API를 이용한 디바이스 위치 확인
비동기 통신으로 가져온 주소 데이터를 가공하여 지도에 마커로 뿌려주었다. 

```java

```
<br>

### 실시간 오류 데이터 감지

```javascript

```
<br>

## 시연 영상 

 {% youtube "https://www.youtube.com/embed/ILHozj9ncG0" %}

## 프로젝트 회고
 