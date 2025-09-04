---
title: Firestore에 마이그레이션한 JSON 데이터로 Google Maps Polygon 그리기(수정중...)
description: 대한민국 시군구 경계 데이터를 Firestore에 마이그레이션하고, 이를 활용하여 Google Maps에 Polygon을 그리는 방법을 다룹니다. 또한, 사용자 위치 기반으로 육각형 polygon을 그리는 Scratch map 구현에 대해 설명합니다.
date: 2025-08-03 10:00:00 +0900
comments: true
categories: [파쿠르 스팟(Parkour Spot in Korea), 2. 기술 및 기능]
tags: [parkourspot, firestore, migration, polygon, googlemap, hexagon]
 
# 태그는 소문자 권장
---

## 대한민국 행정 경계 데이터 활용
Scratch map(스크래치 맵)을 구현하기 위해선 Google Map에 polygon으로 대한민국 경계선을 그려주어야 합니다. 이 경계선은 시, 군, 구, 동을 구분시켜주는 선으로 대한민국 국토교통부 공식 홈페이지에서 데이터 리소스를 제공해주고 있습니다.  
[국토교통부](https://www.data.go.kr/data/15125045/fileData.do?recommendDataYn=Y)

하지만 이 데이터 확장자는 SHP로 되어있어 이를 JSON으로 변환하는 작업이 필요합니다. 또한, 시도 별로 파일이 나누어져 있어 이를 하나로 합치는 작업도 추가로 해주어야 하기 때문에 다소 번거로움이 있습니다. 

감사하게도 깃허브에 대한민국 경계데이터를 geojson으로 변환해주신 분이 계셔서 이를 활용하였습니다. 업데이트로 2025년 최신으로 유지해주셔서 데이터로써 가치가 있습니다. 

[대한민국 행정동 경계 파일](https://github.com/vuski/admdongkor)

먼저 이 데이터를 이용해서 실제로 Google Map에 대한민국 경계가 그려지는지 검증하는 과정이 필요합니다. 빠른 확인을 위해 assets 라이브러리에 35GB 용량의 HangJeongDong_ver20250401.geojson 파일을 넣고 Google Map에 그려지는지 확인해보았습니다. 