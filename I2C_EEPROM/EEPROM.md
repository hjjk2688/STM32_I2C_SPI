# I2C-EEPROM

<img width="600" height="600" alt="F103RB-pin" src="https://github.com/user-attachments/assets/45bb557f-9517-419d-b45c-81a92869bac0" />
<br>

<img width="600" height="400" alt="K24C256" src="https://github.com/user-attachments/assets/19e7704c-dd37-47b5-a35d-39d4bba37158" />
<br>

<img width="800" height="600" alt="I2C_EEPROM_003" src="https://github.com/user-attachments/assets/6f12cb52-ecb1-41d0-a5b5-b02cc4c5c4db" />
<br>
<img width="800" height="600" alt="I2C_EEPROM_001" src="https://github.com/user-attachments/assets/e78b6b0c-52c2-41e7-b09c-bbc37888d74e" />
<br>
<img width="800" height="600" alt="I2C_EEPROM_002" src="https://github.com/user-attachments/assets/b1e7079c-ea43-445a-801f-d66b4933c423" />
<br>

```c
/* USER CODE BEGIN Includes */
#include <stdio.h>
#include <string.h>
/* USER CODE END Includes */
```

```c
/* USER CODE BEGIN PD */
#define K24C256_ADDR        0xA0    // K24C256 I2C 주소 (A0,A1,A2 = 000)
#define EEPROM_PAGE_SIZE    64      // K24C256 페이지 크기 (64 bytes)
#define EEPROM_SIZE         32768   // K24C256 총 크기 (32KB)
#define TEST_ADDRESS        0x0000  // 테스트용 주소
/* USER CODE END PD */
```

```c
/* USER CODE BEGIN PV */
uint8_t i2c_scan_found = 0;
uint8_t eeprom_address = 0;
/* USER CODE END PV */
```

```c
/* USER CODE BEGIN PFP */
void I2C_Scan(void);
HAL_StatusTypeDef EEPROM_Write(uint16_t mem_addr, uint8_t *data, uint16_t size);
HAL_StatusTypeDef EEPROM_Read(uint16_t mem_addr, uint8_t *data, uint16_t size);
void EEPROM_Test(void);
/* USER CODE END PFP */
```

```c
/* USER CODE BEGIN 0 */
#ifdef __GNUC__
/* With GCC, small printf (option LD Linker->Libraries->Small printf
   set to 'Yes') calls __io_putchar() */
#define PUTCHAR_PROTOTYPE int __io_putchar(int ch)
#else
#define PUTCHAR_PROTOTYPE int fputc(int ch, FILE *f)
#endif /* __GNUC__ */

/**
  * @brief  Retargets the C library printf function to the USART.
  * @param  None
  * @retval None
  */
PUTCHAR_PROTOTYPE
{
  /* Place your implementation of fputc here */
  /* e.g. write a character to the USART2 and Loop until the end of transmission */
  if (ch == '\n')
    HAL_UART_Transmit(&huart2, (uint8_t*)"\r", 1, 0xFFFF);
  HAL_UART_Transmit(&huart2, (uint8_t*)&ch, 1, 0xFFFF);

  return ch;
}

/**
  * @brief  I2C 주소 스캔 함수
  * @param  None
  * @retval None
  */
void I2C_Scan(void)
{
  printf("\n=== I2C Address Scan ===\n");
  printf("Scanning I2C bus...\n");

  i2c_scan_found = 0;

  for(uint8_t i = 0; i < 128; i++)
  {
    if(HAL_I2C_IsDeviceReady(&hi2c1, (uint16_t)(i<<1), 3, 5) == HAL_OK)
    {
      printf("Found I2C device at address: 0x%02X (7-bit: 0x%02X)\n", (i<<1), i);
      i2c_scan_found++;

      // K24C256 주소 범위 확인 (0xA0~0xAE)
      if((i<<1) >= 0xA0 && (i<<1) <= 0xAE)
      {
        eeprom_address = (i<<1);
        printf("** K24C256 EEPROM detected at 0x%02X **\n", eeprom_address);
      }
    }
  }

  if(i2c_scan_found == 0)
  {
    printf("No I2C devices found!\n");
  }
  else
  {
    printf("Total %d I2C device(s) found.\n", i2c_scan_found);
  }
  printf("========================\n\n");
}

/**
  * @brief  EEPROM 쓰기 함수
  * @param  mem_addr: 메모리 주소
  * @param  data: 쓸 데이터 포인터
  * @param  size: 데이터 크기
  * @retval HAL_StatusTypeDef
  */
HAL_StatusTypeDef EEPROM_Write(uint16_t mem_addr, uint8_t *data, uint16_t size)
{
  HAL_StatusTypeDef status = HAL_OK;
  uint16_t bytes_to_write;
  uint16_t current_addr = mem_addr;
  uint16_t data_index = 0;

  while(size > 0)
  {
    // 페이지 경계를 고려한 쓰기 크기 계산
    bytes_to_write = EEPROM_PAGE_SIZE - (current_addr % EEPROM_PAGE_SIZE);
    if(bytes_to_write > size)
      bytes_to_write = size;

    // EEPROM에 쓰기
    status = HAL_I2C_Mem_Write(&hi2c1, eeprom_address, current_addr,
                               I2C_MEMADD_SIZE_16BIT, &data[data_index],
                               bytes_to_write, HAL_MAX_DELAY);

    if(status != HAL_OK)
    {
      printf("EEPROM Write Error at address 0x%04X\n", current_addr);
      return status;
    }

    // EEPROM 쓰기 완료 대기 (Write Cycle Time)
    HAL_Delay(5);

    // 다음 쓰기를 위한 변수 업데이트
    current_addr += bytes_to_write;
    data_index += bytes_to_write;
    size -= bytes_to_write;
  }

  return status;
}

/**
  * @brief  EEPROM 읽기 함수
  * @param  mem_addr: 메모리 주소
  * @param  data: 읽을 데이터 포인터
  * @param  size: 데이터 크기
  * @retval HAL_StatusTypeDef
  */
HAL_StatusTypeDef EEPROM_Read(uint16_t mem_addr, uint8_t *data, uint16_t size)
{
  return HAL_I2C_Mem_Read(&hi2c1, eeprom_address, mem_addr,
                          I2C_MEMADD_SIZE_16BIT, data, size, HAL_MAX_DELAY);
}

/**
  * @brief  EEPROM 테스트 함수
  * @param  None
  * @retval None
  */
void EEPROM_Test(void)
{
  if(eeprom_address == 0)
  {
    printf("EEPROM not detected! Cannot perform test.\n\n");
    return;
  }

  printf("=== EEPROM Test ===\n");

  // 테스트 데이터 준비
  char write_data[] = "Hello, STM32F103 with K24C256 EEPROM!";
  uint8_t read_data[100] = {0};
  uint16_t data_len = strlen(write_data);

  printf("Test Address: 0x%04X\n", TEST_ADDRESS);
  printf("Write Data: \"%s\" (%d bytes)\n", write_data, data_len);

  // EEPROM에 데이터 쓰기
  printf("Writing to EEPROM...\n");
  if(EEPROM_Write(TEST_ADDRESS, (uint8_t*)write_data, data_len) == HAL_OK)
  {
    printf("Write successful!\n");
  }
  else
  {
    printf("Write failed!\n");
    return;
  }

  // 잠시 대기
  HAL_Delay(10);

  // EEPROM에서 데이터 읽기
  printf("Reading from EEPROM...\n");
  if(EEPROM_Read(TEST_ADDRESS, read_data, data_len) == HAL_OK)
  {
    printf("Read successful!\n");
    printf("Read Data: \"%s\" (%d bytes)\n", (char*)read_data, data_len);

    // 데이터 비교
    if(memcmp(write_data, read_data, data_len) == 0)
    {
      printf("** Data verification PASSED! **\n");
    }
    else
    {
      printf("** Data verification FAILED! **\n");
    }
  }
  else
  {
    printf("Read failed!\n");
  }

  printf("===================\n\n");

  // 추가 테스트: 숫자 데이터
  printf("=== Number Data Test ===\n");
  uint8_t num_write[10] = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9};
  uint8_t num_read[10] = {0};
  uint16_t num_addr = TEST_ADDRESS + 100;

  printf("Writing numbers 0-9 to address 0x%04X...\n", num_addr);
  if(EEPROM_Write(num_addr, num_write, 10) == HAL_OK)
  {
    HAL_Delay(10);

    if(EEPROM_Read(num_addr, num_read, 10) == HAL_OK)
    {
      printf("Write Data: ");
      for(int i = 0; i < 10; i++) printf("%d ", num_write[i]);
      printf("\n");

      printf("Read Data:  ");
      for(int i = 0; i < 10; i++) printf("%d ", num_read[i]);
      printf("\n");

      if(memcmp(num_write, num_read, 10) == 0)
      {
        printf("** Number test PASSED! **\n");
      }
      else
      {
        printf("** Number test FAILED! **\n");
      }
    }
  }
  printf("========================\n\n");
}
/* USER CODE END 0 */
```

```c
  /* USER CODE BEGIN 2 */
  printf("\n\n");
  printf("========================================\n");
  printf("  STM32F103 I2C EEPROM K24C256 Test    \n");
  printf("  System Clock: 64MHz                  \n");
  printf("  I2C Speed: 100kHz                    \n");
  printf("========================================\n");

  // I2C 주소 스캔
  I2C_Scan();

  // EEPROM 테스트
  EEPROM_Test();

  printf("Test completed. Entering main loop...\n\n");
  /* USER CODE END 2 */
```

```c
  /* Infinite loop */
  /* USER CODE BEGIN WHILE */
  uint32_t loop_count = 0;

  while (1)
  {
    /* USER CODE END WHILE */

	    /* USER CODE BEGIN 3 */

	    // 10초마다 상태 출력
	    if(loop_count % 1000 == 0)
	    {
	      printf("System running... Loop count: %lu\n", loop_count/1000);

	      HAL_GPIO_TogglePin(LD2_GPIO_Port, LD2_Pin);
	    }

	    loop_count++;
	    HAL_Delay(10);
	  }
	  /* USER CODE END 3 */
```

## 2개의 Addredd 관련 이슈

```
=== I2C Address Scan ===
Scanning I2C bus...
Found I2C device at address: 0xA0 (7-bit: 0x50)
** K24C256 EEPROM detected at 0xA0 **
Found I2C device at address: 0xB0 (7-bit: 0x58)
Total 2 I2C device(s) found.
========================
```

### 2개의 블럭이 서로 연관이 있는것인지, 각각의 영역을 다 사용할 수 있는것인지 확인.

```
========================================
  STM32F103 I2C EEPROM K24C256 Test
  System Clock: 64MHz
  I2C Speed: 100kHz
========================================
=== I2C Address Scan ===
Scanning I2C bus...
Found I2C device at address: 0xA0 (7-bit: 0x50)
** K24C256 EEPROM detected at 0xA0 **
Found I2C device at address: 0xB0 (7-bit: 0x58)
Total 2 I2C device(s) found.
========================
=== EEPROM Test ===
Test Address: 0x0000
Write Data: "Hello, STM32F103 with K24C256 EEPROM!" (37 bytes)
Writing to EEPROM...
Write successful!
Reading from EEPROM...
Read successful!
Read Data: "Hello, STM32F103 with K24C256 EEPROM!" (37 bytes)
** Data verification PASSED! **
===================
=== Number Data Test ===
Writing numbers 0-9 to address 0x0064...
Write Data: 0 1 2 3 4 5 6 7 8 9
Read Data:  0 1 2 3 4 5 6 7 8 9
** Number test PASSED! **
========================
=== Testing 0xB0 Device Identity ===
Checking if 0xB0 is an extended memory block...
Test 1: Writing to 0xB0 at address 0x0100...
  Write to 0xB0: SUCCESS
Test 2: Reading from 0xB0 at address 0x0100...
  Read from 0xB0: SUCCESS
  Write Data: "Block1Test"
  Read Data:  "Block1Test"
** 0xB0 IS A VALID EEPROM BLOCK! **
** Your chip is likely 64KB (512Kbit), not 32KB! **
** Block 0 (0xA0): 32KB **
** Block 1 (0xB0): 32KB **
** Total: 64KB available! **
Test 4: Cross-check with 0xA0...
  0xB0 read at 0x0200: SUCCESS
  0xA0 wrote: "CrossCheck"
  0xB0 read:  "Block1Test"
** 0xA0 and 0xB0 have DIFFERENT data! **
** They are INDEPENDENT memory blocks **
** Total 64KB confirmed! **
====================================
Test completed. Entering main loop...
System running... Loop count: 0
```

```
Write to 0xB0: SUCCESS
Read from 0xB0: SUCCESS
데이터 일치 확인
```
→ **0xB0은 진짜 메모리 블럭!**

### ✅ **Test 4: 독립된 메모리 확인 (결정적 증거!)**
```
0xA0에 "CrossCheck" 쓰기
0xB0에서 읽기 → "Block1Test" (다른 데이터!)
```
→ **0xA0과 0xB0은 완전히 독립된 32KB 블럭!**

## 실제 칩 구조:
```
┌─────────────────────────────────┐
│      64KB (512Kbit) EEPROM      │
├─────────────────┬───────────────┤
│   Block 0       │   Block 1     │
│   0xA0          │   0xB0        │
│   32KB          │   32KB        │
│ 0x0000~0x7FFF   │ 0x0000~0x7FFF │
└─────────────────┴───────────────┘
```

  * 테스트 1: EEPROM처럼 동작하는지 확인
  * 테스트2 : 읽기 시도
  * 테스트 3: 다른 프로토콜 시도
  * 크로스 체크: 0xA0와 0xB0가 같은 메모리를 공유하는지 확인
  * 0xA0에 특별한 데이터 쓰기
  * 0xB0의 같은 주소에서 읽기 시도

```c
void Test_0xB0_Device(void)
{
  printf("=== Testing 0xB0 Device Identity ===\n");
  printf("Checking if 0xB0 is an extended memory block...\n\n");
  
  uint8_t write_data[] = "Block1Test";
  uint8_t read_data[20] = {0};
  uint16_t test_addr = 0x0100;
  HAL_StatusTypeDef status;
  
  // 테스트 1: EEPROM처럼 동작하는지 확인
  printf("Test 1: Writing to 0xB0 at address 0x%04X...\n", test_addr);
  status = HAL_I2C_Mem_Write(&hi2c1, 0xB0, test_addr, 
                             I2C_MEMADD_SIZE_16BIT, write_data, 
                             strlen((char*)write_data), HAL_MAX_DELAY);
  
  if(status == HAL_OK)
  {
    printf("  Write to 0xB0: SUCCESS\n");
    HAL_Delay(10);
    
    // 읽기 시도
    printf("Test 2: Reading from 0xB0 at address 0x%04X...\n", test_addr);
    status = HAL_I2C_Mem_Read(&hi2c1, 0xB0, test_addr,
                              I2C_MEMADD_SIZE_16BIT, read_data,
                              strlen((char*)write_data), HAL_MAX_DELAY);
    
    if(status == HAL_OK)
    {
      printf("  Read from 0xB0: SUCCESS\n");
      printf("  Write Data: \"%s\"\n", write_data);
      printf("  Read Data:  \"%s\"\n", read_data);
      
      if(memcmp(write_data, read_data, strlen((char*)write_data)) == 0)
      {
        printf("\n** 0xB0 IS A VALID EEPROM BLOCK! **\n");
        printf("** Your chip is likely 64KB (512Kbit), not 32KB! **\n");
        printf("** Block 0 (0xA0): 32KB **\n");
        printf("** Block 1 (0xB0): 32KB **\n");
        printf("** Total: 64KB available! **\n");
      }
      else
      {
        printf("\n** Data mismatch - 0xB0 behavior unclear **\n");
      }
    }
    else
    {
      printf("  Read from 0xB0: FAILED\n");
      printf("** 0xB0 accepts write but not read - unusual device **\n");
    }
  }
  else
  {
    printf("  Write to 0xB0: FAILED\n");
    
    // 테스트 3: 다른 프로토콜 시도
    printf("\nTest 3: Trying different access methods...\n");
    
    // 8비트 주소로 시도
    status = HAL_I2C_Mem_Read(&hi2c1, 0xB0, 0x00,
                              I2C_MEMADD_SIZE_8BIT, read_data, 8, 1000);
    if(status == HAL_OK)
    {
      printf("  8-bit address read: SUCCESS\n");
      printf("  Data: ");
      for(int i=0; i<8; i++) printf("%02X ", read_data[i]);
      printf("\n");
      printf("** 0xB0 is likely an RTC, sensor, or other I2C device **\n");
    }
    else
    {
      // 단순 수신 시도
      status = HAL_I2C_Master_Receive(&hi2c1, 0xB0, read_data, 8, 1000);
      if(status == HAL_OK)
      {
        printf("  Simple receive: SUCCESS\n");
        printf("  Data: ");
        for(int i=0; i<8; i++) printf("%02X ", read_data[i]);
        printf("\n");
        printf("** 0xB0 responds but protocol unclear **\n");
      }
      else
      {
        printf("  All access methods: FAILED\n");
        printf("** 0xB0 detected but not accessible **\n");
        printf("** Possible ghost address or bus issue **\n");
      }
    }
  }
  
  // 크로스 체크: 0xA0와 0xB0가 같은 메모리를 공유하는지 확인
  printf("\nTest 4: Cross-check with 0xA0...\n");
  memset(read_data, 0, sizeof(read_data));
  
  // 0xA0에 특별한 데이터 쓰기
  uint8_t marker[] = "CrossCheck";
  if(EEPROM_Write(0x0200, marker, strlen((char*)marker)) == HAL_OK)
  {
    HAL_Delay(10);
    
    // 0xB0의 같은 주소에서 읽기 시도
    status = HAL_I2C_Mem_Read(&hi2c1, 0xB0, 0x0200,
                              I2C_MEMADD_SIZE_16BIT, read_data,
                              strlen((char*)marker), HAL_MAX_DELAY);
    
    if(status == HAL_OK)
    {
      printf("  0xB0 read at 0x0200: SUCCESS\n");
      printf("  0xA0 wrote: \"%s\"\n", marker);
      printf("  0xB0 read:  \"%s\"\n", read_data);
      
      if(memcmp(marker, read_data, strlen((char*)marker)) == 0)
      {
        printf("\n** 0xA0 and 0xB0 share SAME memory! **\n");
        printf("** 0xB0 is an ALIAS of 0xA0 - same 32KB chip **\n");
        printf("** Your chip reports multiple addresses **\n");
      }
      else
      {
        printf("\n** 0xA0 and 0xB0 have DIFFERENT data! **\n");
        printf("** They are INDEPENDENT memory blocks **\n");
        printf("** Total 64KB confirmed! **\n");
      }
    }
    else
    {
      printf("  0xB0 read at 0x0200: FAILED\n");
      printf("** Cannot determine relationship **\n");
    }
  }
  
  printf("====================================\n\n");
}
```
