# [권나현] 개인 개발 보고서

**프로젝트명:** Dear Carmate – 중고차 계약 관리 서비스  
**팀명:** 1팀  
**작성자:** 권나현  
**작성일:** 2025.08.12  
 
## 1. 프로젝트 개요

### 1-1. 프로젝트기간 : 7월 22일 ~ 8월 13일

### 1-2. 목적:
- 중고차 판매 과정에서 발생하는 차량·고객·계약 관리 업무를 통합 관리하고, 계약 진행 상황을 시각적으로 확인할 수 있도록 지원하는 서비스입니다.

### 1-3. 주요 기능:
- 차량 등록·수정·삭제·조회
- 유저 관리 (회원관리)
- 계약 관리 (목록 조회)

## 2. 주요 개발 내역

### 백엔드 (Express + Prisma)
- Prisma ORM을 활용한 DB 모델(차량) 설계 및 마이그레이션
- 회원가입 API 구현 및 인증 미들웨어 적용
- 차량 CRUD API 개발
- 차량 등록 시 type 기본값 설정, 사고 횟수 0 허용 처리
- 차량 삭제 시 Prisma 트랜잭션으로 연관된 계약 데이터 Soft Delete 처리
- 차량 제조사/모델(차종) API 구현
- enum + 매핑 객체 기반 Manufacturer/Model 정의
- 프론트에서 바로 선택 가능하도록 /cars/models API 제공
- 계약 목록 상태별 조회 API 개선
- 계약 상태 필터링 로직 보완
- contractDraft 상태 추가
- 차량 검색 기능 개선
- 차량 번호·차종·차량 ID 기반 조회 기능 구현
- 유저 프로필 이미지 생성
- 차량 이미지 생성

### 프론트엔드/테스트
- Postman을 이용한 백엔드 API 단위 테스트 및 통합 테스트
- 프론트 연동 후 차량 등록/수정/조회/삭제, 계약 조회 기능 검증, 유저 회원가입
- 필드 누락, 타입 불일치, 상태 필터 오류 등 디버깅 및 수정

## 3. 개발 이슈 및 해결 내역
| 이슈                                | 원인                      | 해결 방법                                       |
| --------------------------------- | ----------------------- | ------------------------------------------- |
| 차량 등록 시 사고횟수 0 입력 시 필수값 누락 처리됨    | `0`이 falsy 값으로 인식됨      | `data.accidentCount === undefined` 조건으로 수정  |
| 프론트에서 `/cars/models` 호출 시 데이터 미노출 | DB 기반 조회만 구현됨           | 백엔드에서 `Manufacturer`·`Model` 매핑 객체 기반 응답 생성 |
| 차량 삭제 시 계약 데이터 남음                 | 차량 삭제 로직과 계약 삭제 로직이 분리됨 | Prisma 트랜잭션으로 차량+계약 Soft Delete 동시 처리       |
| 차량 검색 시 잘못된 결과 반환                 | ID 조회만 지원               | 차량번호·차종·차량 ID 모두 지원하는 조건문 추가                |
| 계약 상태별 조회 누락 (`contractDraft`)    | 상태 배열에 해당 값 없음          | 상태 배열에 `contractDraft` 추가                   |

## 4. 성과 및 배운 점
- Prisma 트랜잭션을 활용한 연관 데이터 처리 경험을 하였다.
- 타입스크립트에서 enum + 매핑 객체를 사용하여 구조적 데이터 관리 방법 습득하였다.
- 프론트와 백엔드 API 스펙 불일치 시, 백엔드에서 대응하는 방안 설계를 경험 하였다.
- 필수값 검증 로직에서 0 값 허용 처리와 같은 예외 케이스 대응 역량 향상 되었다.
- 상태 필터링, 조건부 검색 등 동적 쿼리 작성 경험 하였다.

## 5. 향후 과제 및 개선 방향
- 차량/계약 데이터 삭제 시 완전 삭제 옵션 추가
- 계약 등록 시 기본 미팅 일정 자동 생성 로직 구현
- 차량 모델 데이터 DB 테이블화 및 관리자 페이지에서 직접 관리 가능하게 개선
- 테스트 코드(Jest) 작성으로 기능 안정성 확보

## 6. 구체적 개발 예시
- 차량 삭제 시 계약 동시 삭제 (Prisma 트랜잭션)
```ts
export const deleteCarCascade = async (carId: bigint, companyId: bigint) => {
  return prisma.$transaction(async (tx) => {
    // 1) 차량 소유/존재 확인
    const car = await tx.car.findFirst({
      where: { id: carId, companyId, isDeleted: false },
      select: { id: true },
    });
    if (!car) throw new NotFoundError('존재하지 않거나 이미 삭제된 차량입니다.');

    // 계약 문서 삭제
    await tx.contractDocument.deleteMany({
      where: { contract: { carId } },
    });

    // 미팅 알람 삭제
    await tx.alarm.deleteMany({
      where: { meeting: { contract: { carId } } },
    });

    // 미팅 삭제
    await tx.meeting.deleteMany({
      where: { contract: { carId } },
    });

    // 계약 소프트 삭제
    await tx.contract.updateMany({
      where: { carId },
      data: { isDeleted: true, deletedAt: new Date() },
    });

    // 차량 이미지 실제 삭제
    await tx.carImage.deleteMany({
      where: { carId },
    });

    // 차량 소프트 삭제
    await tx.car.update({
      where: { id: carId },
      data: { isDeleted: true, deletedAt: new Date() },
    });

    return { id: Number(carId) };
  });
};

```

- 사고 횟수 0 허용 처리
```ts
if (
  !data.carNumber ||
  !data.manufacturer ||
  !data.model ||
  !data.type ||
  data.price === undefined || data.price === null ||
  data.accidentCount === undefined || data.accidentCount === null
) {
  return res.status(400).json({ message: '필수 값이 누락되었습니다.' });
}
```
<img width="431" height="731" alt="스크린샷 2025-08-12 092629" src="https://github.com/user-attachments/assets/8a5e1210-2bd1-4a4a-a871-176801fd62da" />


- 차량 모델/제조사 매핑 응답
```ts
export const getCarModels = (_req: Request, res: Response) => {
  const manufacturers: Manufacturer[] = Object.keys(manufacturerModels) as Manufacturer[];
  const result = manufacturers.map((m) => ({
    manufacturer: m,
    models: manufacturerModels[m],
  }));
  res.status(200).json({ data: result });
};
```
<img width="431" height="899" alt="스크린샷 2025-08-08 162801" src="https://github.com/user-attachments/assets/f93d53d7-5c52-4e8e-983f-a945122ca990" />
<img width="431" height="896" alt="스크린샷 2025-08-08 162807" src="https://github.com/user-attachments/assets/c2600122-a32e-4c7d-99e9-ebe052aa3e1d" />


