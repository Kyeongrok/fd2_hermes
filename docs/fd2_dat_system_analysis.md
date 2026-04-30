# FD2 DAT 파일 시스템 분석 보고서

## 개요
FD2 게임은 통합 DAT 파일 포맷을 사용하여 모든 리소스를 저장하며, 모든 파일의 매직 넘버 헤더는 "LLLLLL"입니다.

## 공통 인덱스 포맷
디컴파일된 함수 `sub_111BA` (주소 0x111BA) 분석 결과:

```c
// DAT 파일 로드 함수
// 파라미터: filename, old_buffer_ptr, index
BYTE* load_dat_entry(const char* filename, BYTE* old_ptr, int index) {
    if (old_ptr) free(old_ptr);
    
    FILE* f = fopen(filename, "rb");
    if (!f) {
        printf("\n\n File not found %s!!! \n\n", filename);
        exit(1);
    }
    
    // 인덱스 테이블 항목 읽기
    fseek(f, 4 * index + 6, SEEK_SET);  // 6바이트 헤더 건너뜀
    DWORD start, end;
    fread(&start, 4, 1, f);
    fread(&end, 4, 1, f);
    
    DWORD size = end - start;
    BYTE* buffer = malloc(size);
    if (!buffer) {
        printf("Out of Memory at Load %s Number:%d!!\n", filename, index);
        exit(1);
    }
    
    fseek(f, start, SEEK_SET);
    fread(buffer, 1, size, f);
    fclose(f);
    
    return buffer;
}
```

## 파일 목록

| 파일 | 크기 | 항목 수 | 주요 용도 |
|------|------|---------|-----------|
| FIGANI.DAT | 15.3 MB | 204 | 캐릭터/오브젝트 애니메이션 |
| FDSHAP.DAT | 3.56 MB | 33 | 도형/스프라이트 데이터 |
| FDOTHER.DAT | 3.23 MB | 52+ | 기타 리소스(팔레트/아이콘 등) |
| ANI.DAT | 2.44 MB | 5 | 장면 애니메이션 |
| DATO.DAT | 1.98 MB | 68 | 게임 UI 요소 |
| BG.DAT | 624 KB | 28 | 배경 이미지 |
| FDFIELD.DAT | 243 KB | 50 | 필드/레벨 데이터 |
| TAI.DAT | 94.9 KB | 28 | 타일셋 |
| FDTXT.DAT | 120 KB | 17 | 텍스트 데이터 |
| FDMUS.DAT | 80.4 KB | 10 | 음악 데이터 |

## FDOTHER.DAT 상세 분석

### 사용 인덱스 통계 (디컴파일 코드 기반)
| 인덱스 | 호출 함수 | 용도 추정 |
|--------|-----------|-----------|
| 0 | sub_10010 | 주 팔레트 |
| 1 | - | 아이콘 인덱스 테이블 (24 서브항목) |
| 15 | sub_10652 | 장면 9, 24, 25 리소스 |
| 16-17 | sub_10652 | 장면 21, 22, 27 스프라이트 |
| 42 | sub_10652 | 장면 23 배경 |
| 55 | sub_10652 | 장면 28, 29 리소스 |
| 56 | sub_31C49 | 게임 UI |
| 64 | sub_1A7BD | - |
| 80 | sub_1D4CB | - |
| 95-102 | 여러 함수 | 게임 오버 화면 |

### 팔레트 항목 (768바이트 = 256색 × 3)
- 항목 0: 주 팔레트 (게임 시작 시 로드)
- 항목 8: 장면 팔레트
- 항목 57, 76, 99, 101, 102: 각 장면 팔레트

### DOS 팔레트 포맷 변환
```python
def convert_dos_palette(data):
    """DOS 6비트 팔레트를 8비트 RGB로 변환"""
    colors = []
    for i in range(0, len(data), 3):
        r = data[i] * 4    # 0-63 -> 0-252
        g = data[i+1] * 4
        b = data[i+2] * 4
        colors.append((r, g, b))
    return colors
```

## RLE 디코딩 포맷

함수 `sub_4E98D`에서 RLE 디코딩을 구현합니다:

### 데이터 헤더
```
struct RLEHeader {
    WORD width;      // 너비 (픽셀)
    WORD height;     // 높이 (행)
    BYTE data[];     // RLE 인코딩 데이터
};
```

### 제어 바이트 인코딩
| 바이트 범위 | 동작 | 파라미터 해석 |
|------------|------|--------------|
| 0x00-0x3F | 픽셀 건너뜀 | count = (byte >> 2) + 1 |
| 0x40-0x7F | 바이트 복사 | count = (byte >> 2) + 1, 소스에서 읽기 |
| 0x80-0xBF | 픽셀 채우기 | count = (byte >> 2) + 1, 다음 바이트로 채움 |
| 0xC0-0xFF | 교번 채우기 | count = (byte >> 2) + 1, 픽셀 쌍 교번 |

### 디코딩 의사코드
```c
void decode_rle(BYTE* src, BYTE* dst, int width, int height) {
    int remaining = width;
    while (height > 0) {
        BYTE ctrl = *src++;
        int count = (ctrl >> 2) + 1;
        
        if (ctrl & 0x80) {        // 0x80-0xFF
            if (ctrl & 0x40) {    // 0xC0-0xFF: 교번 채우기
                BYTE val = *src++;
                for (int i = 0; i < count; i++) {
                    dst[0] = val;
                    dst[1] = val;
                    dst += 2;
                }
            } else {              // 0x80-0xBF: 단색 채우기
                BYTE val = *src++;
                memset(dst, val, count);
                dst += count;
            }
        } else if (ctrl & 0x40) { // 0x40-0x7F: 건너뜀
            dst += count;
        } else {                  // 0x00-0x3F: 복사
            memcpy(dst, src, count);
            src += count;
            dst += count;
        }
        
        remaining -= count;
        if (remaining <= 0) {
            remaining = width;
            height--;
        }
    }
}
```

## 장면 로드 흐름

`sub_2CF30` 분석으로 확인된 장면 로드:

1. **배경 로드 (BG.DAT)** - 장면 번호에 따른 해당 인덱스 로드
2. **타일 로드 (TAI.DAT)** - 동일 인덱스
3. **애니메이션 로드 (FIGANI.DAT)** - 캐릭터 애니메이션 프레임
4. **디코딩 렌더링** - RLE 디코딩 후 버퍼에 렌더링

### 장면 인덱스 매핑
```c
switch (scene_id) {
    case 24: bg_index = 15; break;
    case 28: bg_index = 20; break;
    case 29: bg_index = 13; break;
    default: bg_index = 18; break;
}
```

## 핵심 함수 주소 테이블

| 주소 | 이름 | 기능 |
|------|------|------|
| 0x111BA | load_dat_entry | DAT 파일 범용 로드 |
| 0x4E98D | decode_rle | RLE 디코딩 |
| 0x10652 | load_scene_resources | 장면 리소스 로드 |
| 0x2CF30 | init_scene | 장면 초기화 |
| 0x31C49 | game_loop_init | 게임 메인 루프 초기화 |
| 0x10010 | load_savegame | 세이브 파일 로드 |
| 0x25977 | render_sprite | 스프라이트 렌더링 |

## 다음 작업

1. **모든 팔레트 추출** - 표준 포맷으로 변환
2. **서브인덱스 리소스 디코딩** - 항목 1, 96의 서브 구조 분석
3. **리소스 매핑 테이블 구축** - 장면 인덱스와 파일 인덱스 대응
4. **이미지 내보내기 구현** - PNG/BMP 포맷 변환기

## 참고 파일

- 디컴파일 코드: `/home/yinming/fd2_dat2/tools/export-for-ai/decompile/`
- 함수 인덱스: `/home/yinming/fd2_dat2/tools/export-for-ai/function_index.txt`
- 문자열 테이블: `/home/yinming/fd2_dat2/tools/export-for-ai/strings.txt`
