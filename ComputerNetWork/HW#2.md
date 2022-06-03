# 컴퓨터네트워크 실습2 결과 보고서

> 201911011 컴퓨터과학과 정차미

<br>

  1. 실습 캡쳐


- 수정된 코드 부분
```cpp
else if (arqEvent_checkEventFlag(arqEvent_arqTimeout)) //data TX finished
{
  if (retxCnt > ARQ_MAXRETRANSMISSION){ 
  pc.printf("[WARNING] Failed to send data, max retx cnt reached\n");
  arqEvent_clearEventFlag(arqEvent_arqTimeout);
  main_state = MAINSTATE_IDLE;
  retxCnt = 0;
  break;
}
 …
```


![img](https://imgur.com/vfl5Gkm.png)

<br>

  2. 동작 과정 설명
