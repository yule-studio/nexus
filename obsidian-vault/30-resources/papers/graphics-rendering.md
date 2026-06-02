---
title: "Computer Graphics / Rendering — Papers"
kind: knowledge
status: current
created_at: 2026-06-02T10:00:00+09:00
tags: [resources, papers, graphics, rendering, simulation]
home_hub: resources
---

# Computer Graphics / Rendering — Papers

**[[papers|↑ Papers hub]]**

> 빛 전달·셰이딩·기하 표현·물리 시뮬레이션·신경 렌더링을 아우르는 컴퓨터 그래픽스 핵심 논문 카탈로그.

## 1. 조명 모델 / 셰이딩

- **Illumination for Computer Generated Pictures** (Bui Tuong Phong, CACM, 1975) ⭐
  - 주제: Phong 반사 모델 + per-pixel normal 보간 셰이딩
  - 링크: [dl.acm.org/doi/10.1145/360825.360839](https://dl.acm.org/doi/10.1145/360825.360839)
- **Continuous Shading of Curved Surfaces** (Henri Gouraud, IEEE TC, 1971) ⭐
  - 주제: Gouraud 셰이딩, vertex 단위 명암 보간
  - 링크: [dl.acm.org/doi/10.1109/T-C.1971.223313](https://dl.acm.org/doi/10.1109/T-C.1971.223313)
- **Models of Light Reflection for Computer Synthesized Pictures** (James F. Blinn, SIGGRAPH, 1977) ⭐
  - 주제: Blinn-Phong, microfacet 기반 specular highlight
  - 링크: [dl.acm.org/doi/10.1145/563858.563893](https://dl.acm.org/doi/10.1145/563858.563893)
- **Simulation of Wrinkled Surfaces** (James F. Blinn, SIGGRAPH, 1978) ⭐
  - 주제: bump mapping, normal perturbation으로 표면 디테일
  - 링크: [dl.acm.org/doi/10.1145/800248.507101](https://dl.acm.org/doi/10.1145/800248.507101)
- **A Reflectance Model for Computer Graphics** (Robert L. Cook, Kenneth E. Torrance, SIGGRAPH, 1981) ⭐
  - 주제: Cook-Torrance microfacet BRDF, 물리 기반 specular
  - 링크: [dl.acm.org/doi/10.1145/357290.357293](https://dl.acm.org/doi/10.1145/357290.357293)

## 2. 레이 트레이싱

- **An Improved Illumination Model for Shaded Display** (Turner Whitted, CACM, 1980) ⭐
  - 주제: recursive ray tracing, 반사·굴절·그림자 통합 모델
  - 링크: [dl.acm.org/doi/10.1145/358876.358882](https://dl.acm.org/doi/10.1145/358876.358882)
- **Distributed Ray Tracing** (Robert L. Cook, Thomas Porter, Loren Carpenter, SIGGRAPH, 1984) ⭐
  - 주제: 분산 샘플링으로 motion blur·soft shadow·DOF
  - 링크: [dl.acm.org/doi/10.1145/800031.808590](https://dl.acm.org/doi/10.1145/800031.808590)
- **A 3-Dimensional Representation for Fast Rendering of Complex Scenes** (Steven Rubin, Turner Whitted, SIGGRAPH, 1980)
  - 주제: bounding volume hierarchy 초기 형태
  - 링크: [dl.acm.org/doi/10.1145/965105.807479](https://dl.acm.org/doi/10.1145/965105.807479)

## 3. 전역 조명 / 빛 전달

- **The Rendering Equation** (James T. Kajiya, SIGGRAPH, 1986) ⭐🏆
  - 주제: 모든 렌더링을 통합하는 적분 방정식 + path tracing
  - 링크: [dl.acm.org/doi/10.1145/15922.15902](https://dl.acm.org/doi/10.1145/15922.15902)
- **Modeling the Interaction of Light Between Diffuse Surfaces** (Cindy M. Goral et al., SIGGRAPH, 1984) ⭐
  - 주제: radiosity, diffuse 환경의 에너지 평형
  - 링크: [dl.acm.org/doi/10.1145/800031.808601](https://dl.acm.org/doi/10.1145/800031.808601)
- **Metropolis Light Transport** (Eric Veach, Leonidas J. Guibas, SIGGRAPH, 1997) ⭐
  - 주제: Metropolis-Hastings 변이로 path 샘플링, 어려운 조명에 강건
  - 링크: [dl.acm.org/doi/10.1145/258734.258775](https://dl.acm.org/doi/10.1145/258734.258775) · [graphics.stanford.edu/papers/metro](http://graphics.stanford.edu/papers/metro/) 🆓
- **Bidirectional Estimators for Light Transport** (Eric Veach, Leonidas J. Guibas, EGWR, 1994) ⭐
  - 주제: bidirectional path tracing, eye/light 경로 결합
  - 링크: [graphics.stanford.edu/papers/bdpt](http://graphics.stanford.edu/papers/bdpt/) 🆓
- **A Practical Guide to Global Illumination using Photon Maps** (Henrik Wann Jensen, EGWR, 1996) ⭐
  - 주제: photon mapping, caustics·전역 조명 2-pass
  - 링크: [graphics.stanford.edu/~henrik/papers/ewr7](https://graphics.stanford.edu/~henrik/papers/ewr7/) 🆓
- **Physically Based Rendering: From Theory to Implementation** (Matt Pharr, Wenzel Jakob, Greg Humphreys, 3rd/4th ed.) ⭐📖
  - 주제: PBRT, literate programming 렌더러 레퍼런스
  - 링크: [pbr-book.org](https://www.pbr-book.org/) 🆓

## 4. 텍스처링 / 절차적 생성

- **An Image Synthesizer** (Ken Perlin, SIGGRAPH, 1985) ⭐🏆
  - 주제: Perlin noise, 절차적 텍스처의 기초
  - 링크: [dl.acm.org/doi/10.1145/325334.325247](https://dl.acm.org/doi/10.1145/325334.325247)
- **Pyramidal Parametrics** (Lance Williams, SIGGRAPH, 1983) ⭐
  - 주제: mipmapping, 프리필터링 텍스처 anti-aliasing
  - 링크: [dl.acm.org/doi/10.1145/964967.801126](https://dl.acm.org/doi/10.1145/964967.801126)
- **Reaction-Diffusion Textures** (Greg Turk, SIGGRAPH, 1991)
  - 주제: 반응확산으로 동물 무늬 텍스처 합성
  - 링크: [dl.acm.org/doi/10.1145/122718.122749](https://dl.acm.org/doi/10.1145/122718.122749)

## 5. 기하 / 메시 / 표면 표현

- **Recursively Generated B-Spline Surfaces on Arbitrary Topological Meshes** (Ed Catmull, Jim Clark, CAD, 1978) ⭐
  - 주제: Catmull-Clark 세분화 표면
  - 링크: [dl.acm.org/doi/10.1016/0010-4485%2878%2990110-0](https://dl.acm.org/doi/10.1016/0010-4485%2878%2990110-0)
- **Smooth Subdivision Surfaces Based on Triangles** (Charles Loop, MS thesis, 1987) ⭐
  - 주제: Loop subdivision, 삼각형 메시 평활화
  - 링크: [microsoft.com/en-us/research](https://www.microsoft.com/en-us/research/publication/smooth-subdivision-surfaces-based-on-triangles/) 🆓
- **Marching Cubes: A High Resolution 3D Surface Construction Algorithm** (William E. Lorensen, Harvey E. Cline, SIGGRAPH, 1987) ⭐🏆
  - 주제: 볼륨 데이터에서 isosurface 메시 추출
  - 링크: [dl.acm.org/doi/10.1145/37402.37422](https://dl.acm.org/doi/10.1145/37402.37422)
- **Mesh Optimization** (Hugues Hoppe et al., SIGGRAPH, 1993) ⭐
  - 주제: 에너지 최소화로 메시 단순화·재배치
  - 링크: [dl.acm.org/doi/10.1145/166117.166119](https://dl.acm.org/doi/10.1145/166117.166119)
- **Progressive Meshes** (Hugues Hoppe, SIGGRAPH, 1996) ⭐
  - 주제: LOD·스트리밍을 위한 연속 메시 표현
  - 링크: [dl.acm.org/doi/10.1145/237170.237216](https://dl.acm.org/doi/10.1145/237170.237216)
- **Surface Reconstruction from Unorganized Points** (Hugues Hoppe et al., SIGGRAPH, 1992) ⭐
  - 주제: 점군에서 implicit surface 복원
  - 링크: [dl.acm.org/doi/10.1145/133994.134011](https://dl.acm.org/doi/10.1145/133994.134011)

## 6. 물리 시뮬레이션 / 애니메이션

- **Particle Systems—A Technique for Modeling a Class of Fuzzy Objects** (William T. Reeves, SIGGRAPH, 1983) ⭐
  - 주제: 입자 시스템으로 불·연기·폭발 모델링
  - 링크: [dl.acm.org/doi/10.1145/357306.357320](https://dl.acm.org/doi/10.1145/357306.357320)
- **Stable Fluids** (Jos Stam, SIGGRAPH, 1999) ⭐
  - 주제: semi-Lagrangian advection으로 무조건 안정 유체
  - 링크: [dl.acm.org/doi/10.1145/311535.311548](https://dl.acm.org/doi/10.1145/311535.311548)
- **Position Based Dynamics** (Matthias Müller et al., VRIPhys/J. VCIR, 2007) ⭐
  - 주제: 위치 직접 보정 기반 안정적 변형·천 시뮬
  - 링크: [matthias-research.github.io](https://matthias-research.github.io/pages/publications/posBasedDyn.pdf) 🆓
- **XPBD: Position-Based Simulation of Compliant Constrained Dynamics** (Miles Macklin, Matthias Müller, Nuttapong Chentanez, MIG, 2016)
  - 주제: 시간 스텝 독립 강성의 확장 PBD
  - 링크: [matthias-research.github.io](https://matthias-research.github.io/pages/publications/XPBD.pdf) 🆓
- **Elastically Deformable Models** (Demetri Terzopoulos et al., SIGGRAPH, 1987) ⭐
  - 주제: 연속체 역학 기반 변형체 애니메이션
  - 링크: [dl.acm.org/doi/10.1145/37401.37427](https://dl.acm.org/doi/10.1145/37401.37427)

## 7. 가시성 / 합성 / 디스플레이

- **Compositing Digital Images** (Thomas Porter, Tom Duff, SIGGRAPH, 1984) ⭐
  - 주제: 알파 채널과 Porter-Duff 합성 연산자
  - 링크: [dl.acm.org/doi/10.1145/800031.808606](https://dl.acm.org/doi/10.1145/800031.808606)
- **The Reyes Image Rendering Architecture** (Robert L. Cook, Loren Carpenter, Edwin Catmull, SIGGRAPH, 1987) ⭐
  - 주제: micropolygon 기반 프로덕션 렌더 아키텍처 (RenderMan)
  - 링크: [dl.acm.org/doi/10.1145/37402.37414](https://dl.acm.org/doi/10.1145/37402.37414)
- **Casting Curved Shadows on Curved Surfaces** (Lance Williams, SIGGRAPH, 1978) ⭐
  - 주제: shadow mapping의 기원
  - 링크: [dl.acm.org/doi/10.1145/965139.807402](https://dl.acm.org/doi/10.1145/965139.807402)
- **Sketchpad: A Man-Machine Graphical Communication System** (Ivan E. Sutherland, AFIPS, 1963) ⭐🏆
  - 주제: 대화형 그래픽스·제약 기반 CAD의 효시 (cross-ref [[hci-visualization]])
  - 링크: [dl.acm.org/doi/10.1145/1461551.1461591](https://dl.acm.org/doi/10.1145/1461551.1461591)

## 8. 신경 렌더링 / 방사장 (Neural / Radiance Fields)

- **NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis** (Ben Mildenhall et al., ECCV, 2020) 🆕🏆
  - 주제: MLP로 부피 방사장 인코딩, 신규 시점 합성
  - 링크: [arxiv.org/abs/2003.08934](https://arxiv.org/abs/2003.08934) 🆓 · [matthewtancik.com/nerf](https://www.matthewtancik.com/nerf) 🆓
- **3D Gaussian Splatting for Real-Time Radiance Field Rendering** (Bernhard Kerbl et al., SIGGRAPH/TOG, 2023) 🆕🏆
  - 주제: 명시적 3D Gaussian으로 실시간 고품질 방사장
  - 링크: [dl.acm.org/doi/10.1145/3592433](https://dl.acm.org/doi/10.1145/3592433) · [repo-sam.inria.fr/fungraph/3d-gaussian-splatting](https://repo-sam.inria.fr/fungraph/3d-gaussian-splatting/) 🆓
- **Instant Neural Graphics Primitives with a Multiresolution Hash Encoding** (Thomas Müller et al., SIGGRAPH/TOG, 2022) 🆕🏆
  - 주제: 해시 그리드 인코딩으로 초고속 NeRF 학습
  - 링크: [arxiv.org/abs/2201.05989](https://arxiv.org/abs/2201.05989) 🆓
- **DeepSDF: Learning Continuous Signed Distance Functions for Shape Representation** (Jeong Joon Park et al., CVPR, 2019) 🆕
  - 주제: 신경망으로 implicit SDF 형상 표현
  - 링크: [arxiv.org/abs/1901.05103](https://arxiv.org/abs/1901.05103) 🆓

## 관련
- [[papers]] — Papers hub
- [[hci-visualization]] — 대화형 그래픽스·시각화 (Sketchpad, 디스플레이)
- [[ai-ml-papers]] — 신경 렌더링·미분 가능 그래픽스의 학습 기반
- [[theory-algorithms]] — 가속 구조·기하 알고리즘·Monte Carlo
- [[computer-architecture]] — GPU·실시간 렌더 파이프라인 하드웨어
