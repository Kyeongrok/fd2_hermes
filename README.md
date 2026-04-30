# FD2 리버스 엔지니어링 프로젝트

## 디렉토리 구조

```
fd2_hermes/
├── docs/
│   ├── AFM_FORMAT.md          # AFM 포맷 분석 문서
│   └── afm_animations/        # 디코딩된 애니메이션 파일
│       ├── afm_0.gif
│       ├── afm_1.gif
│       └── ...
├── tools/
│   └── decode_afm_fixed.py    # AFM 디코딩 도구
├── game/
│   └── ANI.DAT                # 게임 데이터
└── README.md
```

## 완료된 작업

### ANI.DAT AFM 디코딩

- ANI.DAT 파일 구조 분석
- AFM 인덱스 및 프레임 포맷 이해
- 10가지 명령 타입 디코딩 (0x00-0x09)
- 9개 AFM 애니메이션을 GIF로 추출 성공

### 주요 발견 사항

1. **인덱스 포맷**: 4바이트 오프셋, 파일 오프셋 6부터 시작
2. **프레임 데이터는 명령 스트림**: 직접 픽셀 데이터가 아닌 명령 시퀀스
3. **명령 0x09**: 프레임 데이터에서 픽셀 버퍼로 복사 (버퍼 내부 복사 아님)
4. **팔레트**: DOS 6비트 포맷, ×4 변환 필요

## 도구 사용법

```bash
cd /home/yinming/fd2_hermes/tools
python3 decode_afm_fixed.py
```

출력 디렉토리: `docs/afm_animations/`

## 다음 단계

- BG.DAT 분석 (배경 이미지)
- FIGANI.DAT 분석 (캐릭터 애니메이션)
- TAI.DAT 분석 (타일 데이터)
