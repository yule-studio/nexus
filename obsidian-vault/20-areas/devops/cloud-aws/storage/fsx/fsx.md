---
title: "Amazon FSx"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T12:15:00+09:00
tags:
  - aws
  - storage
  - file
  - fsx
---

# Amazon FSx

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | FSx hub |

**[[../storage|↑ Storage]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

매니지드 **파일 시스템 4 종**: Windows / Lustre / NetApp ONTAP / OpenZFS. EFS (Linux NFS) 가 안 되는 시나리오 전부 커버.

## 2. 왜 / 언제

| 종류 | 언제 |
| --- | --- |
| **FSx for Windows File Server** | Windows SMB / NTFS / Active Directory |
| **FSx for Lustre** | HPC / ML 학습의 병렬 파일 시스템 (S3 와 연동) |
| **FSx for NetApp ONTAP** | 온프레 NetApp 사용자, multi-protocol (NFS + SMB) |
| **FSx for OpenZFS** | snapshot / clone / compression, ZFS 익숙한 곳 |

## 3. 함정 / 비용

- EFS 보다 보통 비쌈
- 일반 Linux NFS = **EFS** 가 디폴트 (FSx 아님)
- Lustre 는 S3 backed → S3 객체를 파일 시스템처럼 (ML 학습 자주 활용)
- ONTAP / OpenZFS 는 snapshot / clone 으로 dev 환경 빠르게 복제 가능

## 4. 관련

- [[../storage|↑ Storage]]
- [[../efs/efs]] — Linux NFS 기본
- [[../s3/s3]]
