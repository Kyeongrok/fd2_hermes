# FDICON.B24 포맷 분석 (수정판)

## 개요
FDICON.B24는 FD2 게임의 아이콘 리소스 파일로, 140개의 24x24픽셀 아이콘을 포함하며 각 아이콘은 최대 12프레임의 애니메이션을 가집니다.

## 파일 구조

### 헤더 (6바이트)
```
오프셋 0-1: width (WORD) = 24
오프셋 2-3: height (WORD) = 24
오프셋 4-5: 미상 (버전 또는 예약)
```

### 인덱스 테이블 (오프셋 6 ~ 6726)
- 크기: 6720바이트
- 아이콘당 48바이트 인덱스 데이터
- 총 140개 아이콘 (6720 / 48 = 140)
- 각 인덱스는 13개의 DWORD (4바이트) 오프셋값을 포함

### 인덱스 항목 구조
```
아이콘 인덱스 구조 (48바이트):
  DWORD[0-11]: 12프레임 애니메이션 데이터의 오프셋 주소
  DWORD[12]: 종료 오프셋 (마지막 프레임 크기 계산용)
```

### 데이터 영역 (오프셋 6726 이후)
압축된 아이콘 프레임 데이터를 포함합니다.

## 프레임 데이터 압축 포맷

아이콘 프레임은 RLE (Run-Length Encoding) 압축을 사용합니다:

### 인코딩 규칙
- **리터럴**: 0xFE가 아닌 바이트는 픽셀값으로 직접 사용
- **RLE 마커**: 0xFE는 압축 명령을 나타냄
  - `0xFE 0x80-0xFF [value]`: value를 (cmd - 0x80 + 3)번 반복
  - 예시: `FE C3 05` = 0x05를 (0xC3 - 0x80 + 3 = 68)번 반복

### 디코딩 알고리즘
```python
def decode_icon_frame(data):
    result = []
    i = 0
    while i < len(data) and len(result) < 600:
        b = data[i]
        i += 1
        
        if b == 0xFE and i < len(data):
            cmd = data[i]
            i += 1
            
            if cmd >= 0x80:
                # RLE: 다음 바이트 반복
                count = (cmd - 0x80) + 3
                if i < len(data):
                    val = data[i]
                    i += 1
                    result.extend([val] * count)
            else:
                result.append(cmd)
        else:
            result.append(b)
    
    return bytes(result[:576])  # 24x24 = 576바이트
```

## 팔레트

아이콘은 FDOTHER.DAT의 팔레트를 사용합니다:
- 위치: FDOTHER.DAT 리소스 28
- 오프셋: 0x21D119
- 크기: 768바이트 (256색 x 3채널)
- 포맷: DOS 6비트 RGB (값에 4를 곱해 8비트로 변환)

## Python 추출기

```python
import struct
from PIL import Image

def extract_fdicon_b24(icon_path, palette_path, output_dir):
    """FDICON.B24의 모든 아이콘 추출"""
    
    # 아이콘 파일 로드
    with open(icon_path, 'rb') as f:
        icon_data = f.read()
    
    # 팔레트 로드
    with open(palette_path, 'rb') as f:
        other_data = f.read()
    
    pal_offset = 0x21D119
    pal_data = other_data[pal_offset:pal_offset+768]
    palette = []
    for i in range(256):
        r = min(255, pal_data[i * 3] * 4)
        g = min(255, pal_data[i * 3 + 1] * 4)
        b = min(255, pal_data[i * 3 + 2] * 4)
        palette.extend([r, g, b])
    
    # 인덱스 테이블 파싱
    index_data = icon_data[6:6+6720]
    icon_count = 140
    
    for icon_idx in range(icon_count):
        base = icon_idx * 48
        offsets = []
        for j in range(13):
            val = struct.unpack('<I', index_data[base + j*4:base + j*4 + 4])[0]
            offsets.append(val)
        
        # 첫 번째 프레임 추출
        frame_start = offsets[0]
        frame_end = offsets[1] if offsets[1] > frame_start else frame_start + 600
        
        if frame_start > 6 and frame_start < len(icon_data) - 10:
            frame_data = icon_data[frame_start:frame_end]
            decoded = decode_icon_frame(frame_data)
            
            if len(decoded) == 576:
                img = Image.new('P', (24, 24))
                img.putdata(list(decoded))
                img.putpalette(palette)
                img.save(f'{output_dir}/icon_{icon_idx:03d}.png')

def decode_icon_frame(data):
    """단일 프레임 아이콘 데이터 디코딩"""
    result = []
    i = 0
    while i < len(data) and len(result) < 600:
        b = data[i]
        i += 1
        if b == 0xFE and i < len(data):
            cmd = data[i]
            i += 1
            if cmd >= 0x80:
                count = (cmd - 0x80) + 3
                if i < len(data):
                    val = data[i]
                    i += 1
                    result.extend([val] * count)
            else:
                result.append(cmd)
        else:
            result.append(b)
    return bytes(result[:576])

# 사용 예시
extract_fdicon_b24('game/FDICON.B24', 'game/FDOTHER.DAT', 'extracted/icons')
```

## 추출 결과
- 125개 아이콘 추출 성공
- 저장 경로: `extracted/icons_v2/`
- 미리보기 이미지: `extracted/icons_v2_preview.png`

## 이전 분석과의 차이점

### 이전의 잘못된 분석
1. 오프셋 테이블을 `[padding][offset]` 포맷으로 잘못 판단
2. 실제로는 인덱스 테이블이 아이콘당 48바이트이며, 13개의 오프셋값을 포함
3. 데이터 영역의 `02 00` 클러스터는 다른 코드 데이터이며, 아이콘 포맷과 무관

### 올바른 분석 출처
IDA Pro 디컴파일 코드 `sub_11019` 분석으로 확인:
- `fseek(file, 6, 0)` - 인덱스 테이블은 오프셋 6부터 시작
- `v13[n13] = *(_DWORD *)&v6[48 * a5 + 4 * n13]` - 아이콘당 48바이트 인덱스
- `v14 = v13[12] - v13[0]` - 13번째 오프셋으로 크기 계산

## 아이콘 내용
추출된 아이콘에는 다음이 포함됩니다:
- 게임 UI 요소
- 버튼 및 컨트롤
- 캐릭터 초상화/아이콘
- 아이템 아이콘
- 상태 표시기

## 주의사항
1. 각 아이콘은 최대 12프레임 애니메이션 가능
2. 프레임 크기는 고정되지 않으며 압축 포맷 사용
3. 일부 오프셋값은 파일 범위를 벗어나 유효하지 않을 수 있음
4. 팔레트는 반드시 FDOTHER.DAT에서 로드해야 함
