---
title: Firestore에 마이그레이션한 JSON 데이터로 Google Maps Polygon 그리기(수정중...)
description: 대한민국 시군구 경계 데이터를 Firestore에 마이그레이션하고, 이를 활용하여 Google Maps에 Polygon을 그리는 방법을 다룹니다. 또한, 사용자 위치 기반으로 육각형 polygon을 그리는 Scratch map 구현에 대해 설명합니다.
date: 2025-08-03 10:00:00 +0900
comments: true
categories: [파쿠르 스팟(Parkour Spot in Korea), 2. 기술 및 기능]
tags: [parkourspot, firestore, migration, polygon, googlemap, crud]
image: /assets/img/Product_Lockup_Cloud_Firestore_Vertical_Full_Color.png  

---

## 대한민국 행정 경계 데이터 활용
Scratch map(스크래치 맵)을 구현하기 위해선 Google Map에 polygon으로 대한민국 경계선을 그려주어야 합니다. 이 경계선은 시, 군, 구, 동을 구분시켜주는 선으로 대한민국 국토교통부 공식 홈페이지에서 데이터 리소스를 제공해주고 있습니다. 이를 이용하면 가장 최신 정보이면서 정확한 데이터를 사용할 수 있다는 장점이 있습니다. 

[국토교통부](https://www.data.go.kr/data/15125045/fileData.do?recommendDataYn=Y)

> Scratch Map은 지도에 사용자의 위치를 기록하여 다녀간 발자취를 시각적으로 보여줍니다. 이름처럼 사용자가 지도를 스크래치하여 자신만의 지도를 만들어 간다는 느낌을 줍니다. 사용자가 파쿠르

하지만 이 데이터 확장자는 SHP로 되어있어 이를 JSON으로 변환하는 작업이 필요합니다. 또한, 시도 별로 파일이 나누어져 있어 이를 하나로 합치는 작업도 추가로 해주어야 하기 때문에 다소 번거로움이 있습니다. 

감사하게도 GitHub에 대한민국 경계데이터를 geojson으로 변환해주신 분이 계셔서 이를 활용하였습니다. 업데이트로 2025년 최신으로 유지해주고 있기 때문에 데이터로써 가치가 있다고 생각합니다. 

[대한민국 행정동 경계 파일](https://github.com/vuski/admdongkor)

먼저 이 데이터를 이용해서 실제로 Google Map에 대한민국 경계가 그려지는지 검증하는 과정이 필요합니다. 빠른 확인을 위해 assets 라이브러리에 35GB 용량의 HangJeongDong_ver20250401.geojson 파일을 넣고 Google Map에 그려지는지 확인해보았습니다. 

이때, 고려해야할 사항으로 Google Map은 위도, 경도 순으로 좌표를 받는 반면, geojson 데이터는 경도, 위도 순으로 좌표를 제공한다는 점입니다. 따라서 이를 변환해주는 과정이 필요합니다. 


### GeoJson 데이터 구조 분석 및 모델링
우선 GeoJson 파일의 구조를 알아야합니다. GeoJson 파일은 FeatureCollection으로 시작하며, 그 안에 여러 Feature가 들어있습니다. 각 Feature는 geometry와 properties로 구성되어 있습니다. geometry는 실제 좌표 데이터를 포함하고 있으며, properties는 해당 지역의 이름이나 코드와 같은 추가 정보를 담고 있습니다. 

```json
{
  "type": "Feature",
  "properties": {
    "adm_nm": "지역명",
    "adm_cd": "지역코드",
    "adm_cd2": "확장지역코드",
    "sgg" : "시군구코드",
    "sggnm": "시군구명",
    "sido": "시도코드",
    "sidonm": "시도명",
    "coordinates": [
      [
        [경도, 위도],
        [경도, 위도],
        ...
      ]
    ]
  },
}
    ...
  
```

이를 바탕으로 필요한 내용만 골라서 model을 만들었습니다.

```dart
class Geofeature {
  final String? id;
  final String coordinate; 
  final String adm_nm;     
  final int adm_cd;        
  final int sido;         
  final String sidonm;     
  final int sgg;           
  final String sggnm;      
  final int adm_cd2;      
  
    Geofeature({
    this.id,
    required this.coordinate,
    required this.adm_nm,
    required this.adm_cd,
    required this.sido,
    required this.sidonm,
    required this.sgg,
    required this.sggnm,
    required this.adm_cd2,
  });

factory Geofeature.fromGeoJson(Map<String, dynamic> json) {
    final props = json['properties'] as Map<String, dynamic>;
    final geometry = json['geometry'] as Map<String, dynamic>;

    return Geofeature(
      coordinate: jsonEncode(geometry['coordinates']),
      adm_nm: props['adm_nm'] as String? ?? '',
      adm_cd: int.tryParse(props['adm_cd'] as String? ?? '') ?? 0,
      sido: int.tryParse(props['sido'] as String? ?? '') ?? 0,
      sidonm: props['sidonm'] as String? ?? '',
      sgg: int.tryParse(props['sgg'] as String? ?? '') ?? 0,
      sggnm: props['sggnm'] as String? ?? '',
      adm_cd2:int.parse(props['adm_cd2'] as String? ?? '') ?? 0,
    );
  }
}

```

좌표(coordinate) 데이터는 용량이 크고 멀티폴리곤(multi-polygon) 형태이기 때문에 JSON 문자열로 변환하여 저장하는 방식을 선택했습니다. 왜 싱글 폴리곤이 아닌 멀티 폴리곤으로 구성되어있는 것일까 하는 궁금증이 생겨서 알아보니, 대한민국은 경기도 안에 서울 특별시가 포함되어 있어 이는 단일 폴리곤으로 구성할 수 없다는 것을 알게되었습니다. 즉, 하나의 연결된 선 안에 또 다른 선을 포함시키는 것은 싱글 폴리곤으로 표현 할 수 없습니다.

```dart
// 원본: List<List<List<double>>> (복잡한 배열)
final coordinates = geometry['coordinates']; 

// 직렬화: String (단순한 문자열)
final coordinateString = jsonEncode(coordinates);

```

대한민국 행정동 경계파일이 올바르게 Google Map에 폴리곤으로 그려지는 것을 확인한 후에 Firestore에 마이그레이션을 진행하였습니다.

### GeoJson -> Firestore에 마이그레이션
3.5GB의 GeoJSON 파일을 assets에 포함시키는 것은 비효율적입니다.그 이유는 앱 번들에 포함시킬 경우 앱 크기가 커지는 것 뿐만 아니라, 앱 업데이트 마다 전체 파일을 다시 다운로드 받아야 합니다. firestore에 저장해두면 정정이 필요할 때 마다 바로 반영이 가능합니다.

- geojson_migrator.dart

```dart
Future<void> migrateGeoJsonToFirestore({
  required String geoJsonFilePath,
  required String collectionName,
}) async {
  // 1. GeoJSON 파일 읽기
  String geoJsonString = await _readGeoJsonFile(geoJsonFilePath);

  // 2. JSON 파싱
  Map<String, dynamic> geoJsonData = json.decode(geoJsonString);

  // 3. Features 추출
  List<dynamic> features = geoJsonData['features'] ?? [];

  // 4. 각 Feature를 Geofeature 객체로 변환하고 Firestore에 저장
  for (int i = 0; i < features.length; i++) {
    await _saveGeofeatureToFirestore(features[i], collectionName);
  }
}

Future<void> _saveGeofeatureToFirestore(
    Map<String, dynamic> feature,
    String collectionName
) async {
  // GeoJSON Feature를 Geofeature 객체로 변환
  Geofeature geofeature = Geofeature.fromGeoJson(feature);

  // Firestore에 저장할 데이터로 변환
  Map<String, dynamic> firestoreData = geofeature.toMap();

  // Firestore에 문서 추가
  DocumentReference docRef = await _firestore
      .collection(collectionName)
      .add(firestoreData);
}
```

- Firestore에서 데이터 로드
firebase_service.dart 에서 시도별 데이터 가져오기
```dart
static Future<List<Map<String, dynamic>>> loadDocsBySidoRaw(int sido) async {
  final snapshot = await _firestore
      .collection('dong_features')
      .where('sido', isEqualTo: sido)
      .get();

  return snapshot.docs.map((doc) {
    final data = doc.data();
    return {
      'docId': doc.id,
      'sido': data['sido'],
      'sgg': data['sgg'],
      'coordinates': data['coordinates'],  // JSON 문자열로 저장된 좌표
      'updated_at': data['updated_at'],
      'adm_nm': data['adm_nm'],
    };
  }).toList();
}
```

- 좌표 문자열을 LatLng 객체 리스트로 변환
앞서 말씀드린 것처럼 Google Map은 위도, 경도 순으로 좌표를 받기 때문에 이를 변환해주는 과정이 필요합니다.
GesoJson 표준 : [경도, 위도]
Google Map 표준 : LatLng(위도, 경도)

```dart
static List<LatLng> _parseCoordinates(String jsonStr) {
    try {
      final decoded = json.decode(jsonStr);
      final List<dynamic> rawCoords = decoded[0][0];
      return rawCoords.map<LatLng>((pair) => LatLng(pair[1], pair[0])).toList();
    } catch (e) {
      print('좌표 파싱 오류: $e');
      return [];
    }
  }
```


- 마이그레이션 완료된 Firestore 화면

![마이그레이션완료](/assets/img/firestore1.png)



### 로컬 캐싱 시스템 구현
네트워크 요청을 최소화하기 위해 Drift를 활용한 로컬 캐싱 시스템을 구현했습니다. Firestore에서 데이터를 가져올 때마다 로컬 데이터베이스에 저장하여, 다음 번 요청 시 로컬에서 빠르게 데이터를 불러올 수 있도록 했습니다. 이를 통해 앱의 성능을 크게 향상시킬 수 있었습니다.
1. 캐시 테이블 설계
```dart
@DataClassName('PolygonRow')
class Polygons extends Table {
  TextColumn get docId => text().named('doc_id')();
  IntColumn get sido => integer()();
  IntColumn get sggPrefix => integer().named('sgg_prefix').nullable()();
  TextColumn get coordinatesJson => text().named('coordinates_json')();
  DateTimeColumn get updatedAt => dateTime().named('updated_at')();

  @override
  Set<Column> get primaryKey => {docId};
}
```

2. 캐시 우선 로딩 전략
```dart
Future<Set<Polygon>> fetchAndCachePolygons(int sido) async {
  try {
    // 1) Firebase에서 데이터 가져오기
    final docs = await FirebaseService.loadDocsBySidoRaw(sido);
    
    // 2) Drift에 캐시 저장
    final companions = docs.map((m) {
      final sgg = m['sgg'] as int?;
      final sggPrefix = (sgg == null) ? null : sgg - (sgg % 100);
      return PolygonsCompanion.insert(/* ... */);
    }).toList();
    
    await driftMapService.upsertPolygonsCompanions(companions);
    
    // 3) Polygon 객체로 변환해서 반환
    return _convertToPolygons(docs);
  } catch (e) {
    throw Exception('원격 폴리곤 가져오기 실패: $e');
  }
}
```
## 성능 최적화
### 시도별 분할 로딩
전국 데이터를 한 번에 로드하는 대신, 사용자의 현재 위치를 기반으로 필요한 시도의 데이터만 로드합니다. 
```dart
Future<void> _loadPolygonsForSido(int sido) async {
  try {
    // 1. 로컬 캐시 먼저 확인
    final localRows = await _scratchMapRepository.getPolygonsBySido(sido);
    if (localRows.isNotEmpty) {
      final localPolygons = _scratchMapRepository.convertRowsToPolygons(localRows);
      _updateState(_state.copyWith(polygons: localPolygons));
    }
    
    // 2. 원격에서 최신 데이터 가져와서 업데이트
    final remotePolygons = await _scratchMapRepository.fetchAndCachePolygons(sido);
    _updateState(_state.copyWith(polygons: remotePolygons));
  } catch (e) {
    print('❌ 폴리곤 로드 실패: $e');
  }
}
```

### 인덱스 최적화
데이터베이스 조회 성능을 위해 인덱스를 생성했습니다.

```dart
await customStatement(
  'CREATE INDEX IF NOT EXISTS idx_polygons_sido ON polygons(sido);'
);
await customStatement(
  'CREATE INDEX IF NOT EXISTS idx_polygons_sgg_prefix ON polygons(sgg_prefix);'
);
```

## Hexagon 구현
아래 이미지와 같이 Google 지도에 육각형 모양의 폴리곤을 그리기 위한 과정에 대해 설명합니다.
![hexagon예시1](/assets/img/scratchmapexample.png)
