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

### 배포용 설치 파일 제작
프로그램 배포에 용이하도록 설치 스크립트를 만들었다. Ubuntu 16.04 기준으로 개발하고 설치 매뉴얼에 사용 환경을 명시하였다.   
필수 프로그램인 mysql과 tomcat을 설치하고 설치 파일에 포함된 웹 프로그램 war파일을 웹서버 위치에 압축 해제한 후, 사용자의 ip와 mysql 로그인 정보를 입력받아 jdbc 설정 파일을 수정하였다.

```bash
#!/bin/bash

echo "**********Ready to install TLS program..**********"
sudo apt-get update
echo "**********Installing mysql..**********"
sudo apt-get install mysql-server
echo "**********Please enter the PASSWORD for mysql **root** user!**********"
read userpass
echo "**********Setting Database & Tables..**********"
mysql -uroot -p${userpass} mysql < ./code.sql
mysql -uroot -p${userpass} kopo < ./device.sql
mysql -uroot -p${userpass} kopo < ./err_code.sql
mysql -uroot -p${userpass} kopo < ./error.sql
mysql -uroot -p${userpass} kopo < ./success.sql
mysql -uroot -p${userpass} -e "GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' identified by '${userpass}';"
mysql -uroot -p${userpass} -e "FLUSH PRIVILEGES;"
echo "**********mysql installed**********"
echo "**********Allowing access via external IP..**********"
sudo ufw allow 3306/tcp
sudo sed -i "s/bind-address/#bind-address/" /etc/mysql/mysql.conf.d/mysqld.cnf
sudo sed -i "s/[mysql]/[client]\n default-character-set=utf8\n\n [mysql]\n default-character-set=utf8\n\n [mysqld]\n collation-server = utf8_unicode_ci\n init-connect='SET NAMES utf8'\n character-set-server = utf8/" /etc/mysql/conf.d/mysql.cnf
sudo /etc/init.d/mysql restart
echo "**********Installing tomcat8..**********"
sudo apt-get install tomcat8
echo "**********tomcat8 installed**********"
echo "**********Starting tomcat8**********"
sudo service tomcat8 start
echo "**********Installing TLS monitoring program..**********"
sudo cp gztls.war /var/lib/tomcat8/webapps
echo "**********Unzipping program..please wait**********"
until [ -d /var/lib/tomcat8/webapps/gztls ]
do
	echo "now unzipping..";
	sleep 2;
done
if [ -d /var/lib/tomcat8/webapps/gztls ];
then 
	echo "**********Program unzipped**********"
fi
echo "**********Setting configurations..**********"
echo "**********Please enter your external IP(If you use localhost, enter 127.0.0.1)**********"
read userip
sudo sed -i "s/ipipip/${userip}/" /var/lib/tomcat8/webapps/gztls/WEB-INF/views/header.jsp
sudo sed -i "s/ipipip/${userip}/" /var/lib/tomcat8/webapps/gztls/WEB-INF/spring/root-context.xml
sudo sed -i "s/pwpwpw/${userpass}/" /var/lib/tomcat8/webapps/gztls/WEB-INF/spring/root-context.xml
sudo /etc/init.d/tomcat8 restart
echo "**********Now you can access http://${userip}:8080/gztls **********"
echo "**********For more informations please check README.md**********"
echo "**********THANK YOU**********"
exit
```

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

```javascript
//bymap.jsp
...
$.ajax({
   url : "dlistjsn",
   type : "GET",
   error : function() {
      alert("err");
   }
}).done(function(results){
   ...
   for(i in results){
      set=[];
      set.content=results[i].device_name;
      set.latlng= temp(results[i].device_latitude,results[i].device_longitude);
      arr.push(set);
      centerlat+=results[i].device_latitude;
		centerlng+=results[i].device_longitude;		
   }
   centerlat = centerlat/lnth; //마커들의 중점을 구하여 지도의 중점으로
   centerlng = centerlng/lnth;
   getmap(arr,centerlat,centerlng);
   ...
}
...
function getmap(arr, clat, clng){	
	var mapContainer = document.getElementById('map'), 
	mapOption = { 
	    center: new kakao.maps.LatLng(clat, clng), //중점
	    level: 12
	};		
	var map = new kakao.maps.Map(mapContainer, mapOption); //지도 생성
	var mapTypeControl = new kakao.maps.MapTypeControl();	//지도 디자인 관련 설정
	map.addControl(mapTypeControl, kakao.maps.ControlPosition.TOPRIGHT);
	var zoomControl = new kakao.maps.ZoomControl();
	map.addControl(zoomControl, kakao.maps.ControlPosition.RIGHT);
	
	var positions="["; //인포윈도우에 출력될 내용
	for (i=0; i<arr.length; i++){
		positions += "{content: '"+arr[i].content+"', latlng: "+arr[i].latlng+"},"
	}
	positions+="]";
	console.log("positions",positions);
	positions= eval(positions);		
	
	for (var i = 0; i < positions.length; i ++) { //마커 생성
		var marker = new kakao.maps.Marker({
		    map: map,
		    position: positions[i].latlng
		});		
		
		var infowindow = new kakao.maps.InfoWindow({ //인포윈도우 생성
		    content: positions[i].content
		});		
		kakao.maps.event.addListener(marker, 'mouseover', makeOverListener(map, marker, infowindow)); //이벤트 등록
		kakao.maps.event.addListener(marker, 'mouseout', makeOutListener(infowindow));
	}		
	function makeOverListener(map, marker, infowindow) {
		return function() {
		    infowindow.open(map, marker);
		};
	}		
	function makeOutListener(infowindow) {
		return function() {
		    infowindow.close();
		};
	}
}
...
```
<br>

### 실시간 오류 데이터 감지
ajax long polling 방식으로 실시간 오류 데이터 발생을 모니터링하였다.   

```javascript
$(document).ready(function() {	
(function poll(){
	$.ajax({
		url : "errnowjsn",
		type : "GET",
		success: function(results){
			if(results.length != 0){
				alert("에러가 발생했습니다. 실시간 에러 확인 페이지로 이동합니다.");
				window.location.href="?contentPage=errnow.jsp";
			}
		},
		error : function() {
			alert("err");
		},
		complete: poll,
		timeout: 600000
	});
})();
});
```
<br>

## 시연 영상 
### 설치
 {% youtube "https://youtu.be/lR6mtzKAh7M" %}

### 프로그램 기능
 {% youtube "https://youtu.be/LnaarLEgfaY" %}
## 프로젝트 회고
프론트 위주의 개발은 오랜만이었다. java보다 자유로운 형식 때문에 계획 없이 코드를 짜니 스스로도 파악하기 어려운 복잡한 코드가 되어 다 만들어놓고 처음부터 다시 정리한 페이지가 몇 개나 된다. 코드 정리의 중요성을 체감하는 기회였다.   
테이블 설계부터 개발 환경 설정, 프로그램 설치 파일, 그 외 정의서나 매뉴얼 등 문서 작업까지, 프로그램 배포에 필요한 시작부터 끝까지 온전하게 마친 것은 처음이다. 개발이라 하면 그동안 코드만 짜고 이 프로그램을 어떻게 누군가가 사용할 수 있게 할지 생각해본 적이 없었다. 이번 프로젝트를 통해 얕게나마 그에 대한 고민을 해볼 수 있어 좋았다. 누군가 사용할 수 있다고 생각하니 같은 화면도 다시 한 번 고민하게 되고, 앞으로 개발을 할 때에도 항상 누군가 내 코드를 실제로 사용할 것이라는 마음가짐으로 더 책임감 있게 개발해야겠다.