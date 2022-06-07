# 컴퓨터네트워크 실습2 결과 보고서

> 201911011 컴퓨터과학과 정차미

<br>

  1. 실습 캡쳐


- 수정된 코드 부분
```cpp
#define ARQ_MAXRETRANSMISSION               4 //변수 정의 (10번이 길어서 5번으로 줄임)
…
else if (arqEvent_checkEventFlag(arqEvent_arqTimeout)) //data TX finished
{
  //삽입한 새로 if문 코드
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

<br>

- 실행 결과
![img](https://imgur.com/vfl5Gkm.png)

<br>

  2. 동작 과정 설명

> src에서 dest노드로 "hello" 라는 메시지를 보내는 경우 받는쪽에서 패킷을 2번 받지 못하여 retransmission 2번이 일어나 세번째 전송에 ack수신에 성공하는 경우 모든 동작에 대해서 설명하시오.
> 
- **Step 1.** TX와 RX 번호 입출력
```cpp
pc.printf("------------------ ARQ protocol starts! --------------------------\n");
    arqEvent_clearAllEventFlag();
    
    //source & destination ID setting
    pc.printf(":: ID for this node : ");
    pc.scanf("%d", &endNode_ID);
    pc.printf(":: ID for the destination : ");
    pc.scanf("%d", &dest_ID);
    pc.getc();

    pc.printf("endnode : %i, dest : %i\n", endNode_ID, dest_ID);

    arqLLI_initLowLayer(endNode_ID);
    pc.attach(&arqMain_processInputWord, Serial::RxIrq);
```
endNode_ID = 1, dest_ID = 2

RX는 아직 연결이 안된 상태입니다. 이 상태 그대로 무한 반복문에 진입합니다. pc.attach는 콜백 지정 함수로 첫 번째 인자는 인터럽트가 뜨면 호출될 반환형이 void인 함수의 주소입니다. 함수 `arqMain_processInputWord`의 주소를 첫 번째 인자로 넣었습니다.

- - 함수 `arqMain_processInputWord` 살펴 보기
```cpp

//application event handler : generating SDU from keyboard input
//어플리캐이션 이벤트 핸들러 : 키보드로 입력으로 SDU를 가져오기
void arqMain_processInputWord(void)
{
    //문자를 가져오기
    char c = pc.getc();
    //메인 상태가 IDLE이고 당장 데이터를 보내려는 상태가 아닐 때,
    if (main_state == MAINSTATE_IDLE &&
        !arqEvent_checkEventFlag(arqEvent_dataToSend))
    {   //문자열이 끝났다면 (마지막 문자가 '엔터' 또는 '첫줄로' 개행 문자일 때)
        if (c == '\n' || c == '\r')
        {   //문자열이 종결되었음을 표시해준다.
            originalWord[wordLen++] = '\0';
            //SDU를 dest 노드에 보내기 위해 세팅할 준비를 알린다.
            arqEvent_setEventFlag(arqEvent_dataToSend);
            //사용자가 작성하였던 문자를 화면에 출력해준다.
            pc.printf("word is ready! ::: %s\n", originalWord);
        }
        else
        {   //만약 입력을 끝내려는 상태가 아니라면 문자를 계속 입력받는다.
            originalWord[wordLen++] = c;
            //입력 된 문자열이 배열 크기 초과
            if (wordLen >= ARQMSG_MAXDATASIZE-1)
            {   //강제로 문자열 종료 후 경고문 출력
                originalWord[wordLen++] = '\0';
                arqEvent_setEventFlag(arqEvent_dataToSend);
                pc.printf("\n max reached! word forced to be ready :::: %s\n", originalWord);
            }
        }
    }
}

```
