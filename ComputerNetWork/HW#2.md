# 컴퓨터네트워크 실습2 결과 보고서

> 201911011 컴퓨터과학과 정차미

<br>

  1. 실습 캡쳐


- 수정된 코드 부분
```cpp
#define ARQ_MAXRETRANSMISSION               4 //변수 정의 (10번이 길어서 5번으로 줄임)
…
//삽입한 새로 if문 코드
else if (arqEvent_checkEventFlag(arqEvent_arqTimeout)) //data TX finished
{
  if (retxCnt > ARQ_MAXRETRANSMISSION){ //타이머가 만료되어 타임아웃이 5번 발생하였을 때,
  pc.printf("[WARNING] Failed to send data, max retx cnt reached\n"); //경고 메세지를 출력
  arqEvent_clearEventFlag(arqEvent_arqTimeout); //이벤트를 초기화 해준다
  main_state = MAINSTATE_IDLE; //메인 상태를 IDLE로 변경
  retxCnt = 0; //타임 아웃 카운터를 초기화
  break; //if문 종료
}
//if문 아래에는 기존의 코드가 있습니다
 …
```


![img](https://imgur.com/vfl5Gkm.png)

<br>

  2. 동작 과정 설명
