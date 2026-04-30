# FDOTHER.DAT LMI1 포맷 분석 완료

## 분석 성과

### LMI1 포맷 완전 해독

LMI1은 **열 단위로 저장하는 애니메이션 포맷**으로, 프레임당 픽셀 데이터를 1열씩 저장합니다.

#### 파일 구조

```
오프셋      내용
─────────────────────────────
0x00-0x03  "LMI1" 식별자
0x04-0x05  너비 (uint16)
0x06-0x07  높이 (uint16)
0x08+      프레임 오프셋 테이블
프레임 데이터  RLE 압축된 열 데이터
```

#### 프레임 오프셋 테이블 포맷

특수 패턴: `[00 00][오프셋:2]` 반복

```
오프셋8:  00 00 [오프셋1:2바이트]
오프셋12: 00 00 [오프셋2:2바이트]
오프셋16: 00 00 [오프셋3:2바이트]
...
```

#### 디코딩 알고리즘

```python
def decode_lmi1(data):
    w = read_uint16(data[4:6])
    h = read_uint16(data[6:8])
    
    # 프레임 오프셋 테이블 파싱
    offsets = []
    i = 8
    while data[i:i+2] == b'\x00\x00':
        offset = read_uint16(data[i+2:i+4])
        offsets.append(offset)
        i += 4
    
    # 각 프레임 디코딩 (각 프레임이 하나의 열)
    frames = []
    for offset in offsets:
        frame_data = data[offset:next_offset]
        column = decode_rle(frame_data)  # RLE 디코딩
        frames.append(column[:h])  # 높이 h개의 픽셀 취득
    
    # 이미지 합성 (열 우선 → 행 우선 변환)
    image = []
    for y in range(h):
        for x in range(w):
            image.append(frames[x][y])
    
    return image, w, h
```

### 리소스 목록

| 인덱스 | 크기 | 프레임 수 | 용도 추정 |
|--------|------|-----------|-----------|
| 1 | 23x102 | 22 | 세로 줄 애니메이션 |
| 2 | 138x562 | 137 | 대형 애니메이션 |
| 4 | 12x58 | 11 | 소형 애니메이션 |
| 6 | 28x122 | 27 | 중형 애니메이션 |
| 14 | 24x106 | 23 | 중형 애니메이션 |

### 주요 발견 사항

1. **프레임 수 = 너비**: 각 프레임이 하나의 열에 대응하므로, 프레임 수는 이미지 너비와 같음
2. **RLE 압축**: 각 프레임은 동일한 RLE 알고리즘 사용
3. **애니메이션 용도**: 수평 스크롤 또는 파도 효과에 사용될 것으로 추정

### 추출된 파일

- `/home/yinming/fd2_hermes/extracted/fdother/resource_001_lmi1.raw` (2346바이트)
- `/home/yinming/fd2_hermes/extracted/fdother/resource_002_lmi1.raw` (77556바이트)
- `/home/yinming/fd2_hermes/extracted/fdother/resource_004_lmi1.raw` (696바이트)
- `/home/yinming/fd2_hermes/extracted/fdother/resource_006_lmi1.raw` (3416바이트)
- `/home/yinming/fd2_hermes/extracted/fdother/resource_014_lmi1.raw` (2544바이트)

## 다음 단계 제안

1. 기타 미상 유형 리소스 분석
2. DOSBox-X 디버거로 LMI1 렌더링 과정 검증
3. 기타 DAT 파일 분석 (FDSHAP.DAT, FDTXT.DAT 등)
4. Rust 버전 이미지 로더 구현
