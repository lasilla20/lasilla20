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
            {   //강제로 문자열 종료 후 경고문 출력한다.
                originalWord[wordLen++] = '\0';
                arqEvent_setEventFlag(arqEvent_dataToSend);
                pc.printf("\n max reached! word forced to be ready :::: %s\n", original
                Word);
            }
        }
    }
}

```
여기서 사용자는 hello 를 입력할 것이므로 변수 originalWord는,

originalWord = "hello\0" 입니다.

<br>

- **Step 2.** 메세지 전송하기

아래 코드는 메인 함수의 while 무한 반복 내용입니다.
```cpp
        if (prev_state != main_state)
        {
            debug_if(DBGMSG_ARQ, "[ARQ] State transition from %i to %i\n", prev_state,
            main_state);
            prev_state = main_state;
        }
```
맨 처음 시작 시, prev_state가 0으로 초기화되어 있으므로 protocal을 시작함을 알리면서 prev_state와 main_state를 동기화시키는 작업을 수행합니다. IDLE 상태로 정상 작동 중이니 prev_state 또한 IDLE 상태가 됩니다. 메세지를 전송하기 위해 반복문 안에서 

`case MAINSTATE_IDLE` → `else if arqEvent_checkEventFlag(arqEvent_dataToSend))`

으로 넘어옵니다.

```cpp
else if (arqEvent_checkEventFlag(arqEvent_dataToSend)) //if data needs to be sent (keyboard input)
{ //msg header setting
  //보낼 메세지를 encoding 합니다.
  pduSize = arqMsg_encodeData(arqPdu, originalWord, seqNum, wordLen);
  //dest_ID(2)로 메세지를 전송합시다.
  arqLLI_sendData(arqPdu, pduSize, dest_ID);
  
  //Setting ARQ parameter 
  seqNum = (seqNum + 1)%ARQMSSG_MAX_SEQNUM;
  retxCnt = 0;
  //메세지 출력 문구 (2에게 보내는 시간을 알림)
  pc.printf("[MAIN] sending to %i (seq:%i)\n", dest_ID, (seqNum-1)%ARQMSSG_MAX_SEQNUM);
  //main 상태를 TX로 변경
  main_state = MAINSTATE_TX;
  flag_needPrint = 1;

  wordLen = 0;
  arqEvent_clearEventFlag(arqEvent_dataToSend);
  }
  
```
아직까진 RX가 연결되지 않아도 프로토콜이 원활하게 수행됩니다. 메인 상태가 TX로 변경되었으므로 case IDLE을 빠져나와

`case MAINSTATE_TX` → `else if (arqEvent_checkEventFlag(arqEvent_dataTxDone))`

로 이동합니다.

```cpp
else if (arqEvent_checkEventFlag(arqEvent_dataTxDone)) //data TX finished
{ //ACK으로 메인 상태를 바꾸고 타이머 시작
  main_state = MAINSTATE_ACK;
  arqTimer_startTimer(); //start ARQ timer for retransmission
  
  arqEvent_clearEventFlag(arqEvent_dataTxDone);
  }
```

src 에서는 메세지 전송이 완료되었으나 dest 노드에서 이를 받지 못해 src 또한 ACK를 받지 못하고, 타이머가 만료되고 맙니다. 첫 번째 타임 아웃 이벤트가 발생하였습니다.

`case MAINSTATE_ACK` → `else if (arqEvent_checkEventFlag(arqEvent_arqTimeout))`

으로 이동하여 dest 노드가 패킷을 받지 못하고 타이머가 만료되었을 시 어떠한 동작이 수행되는지 살펴봅니다.

<br>
<br>

- **Step 3.** 메세지 전송 실패

```cpp
else if (arqEvent_checkEventFlag(arqEvent_arqTimeout)) //data TX finished
{ //실습으로 넣은 코드
  //ARQ_MAXRETRANSMISSION = 4; 이므로 타임 아웃이 5번 발생하면
  if (retxCnt > ARQ_MAXRETRANSMISSION){
  //최대 전송 횟수에 도달하여 데이터 전송에 실패하였음을 알린다.
  pc.printf("[WARNING] Failed to send data, max retx cnt reached\n");
  //이벤트를 초기화, 메인 상태를 IDLE로 변경, 타임 아웃 카운터 retxCnt를 초기화
  arqEvent_clearEventFlag(arqEvent_arqTimeout);
  main_state = MAINSTATE_IDLE;
  retxCnt = 0;
  break;
  }
  
  //타임 아웃이 일어났다는 걸 출력하고 retransmission을 수행
  pc.printf("timeout! retransmit\n");
  //다시 데이터를 전송한다.
  arqLLI_sendData(arqPdu, pduSize, dest_ID);
  //Setting ARQ parameter
  //전송 횟수(타임 아웃 카운터) 갱신, 메인 상태를 TX로 변경, 이벤트를 초기화
  retxCnt += 1;
  main_state = MAINSTATE_TX;
  arqEvent_clearEventFlag(arqEvent_arqTimeout);
}
```
아직 첫 번째 타임 아웃이 발생한 것이기 때문에 상단의 if문은 실행되지 않습니다. *timeout! retransmit* 이 1번 출력되고 메인 상태가 TX로 변경되어 다시 `MAINSTATE_TX`로 돌아갑니다.

`case MAINSTATE_TX` → `if (arqEvent_checkEventFlag(arqEvent_dataTxDone))`

타이머를 다시 시작한 후에 ACK로 넘어가면 위의 과정을 반복하다, 세 번째 retransmission 에서 ACK를 수신받게 됩니다.

<br>
<br>

- **Step 3.** 메세지 전송 완료
- - dest node의 경우
```cpp
case MAINSTATE_IDLE: //IDLE state description
  //패킷 수신 이벤트 발생              
  if (arqEvent_checkEventFlag(arqEvent_dataRcvd)) //if data reception event happens
  {
  //Retrieving data info.
  //전송 받은 데이터의 정보를 변수에 저장합니다.
  uint8_t srcId = arqLLI_getSrcId();
  uint8_t* dataPtr = arqLLI_getRcvdDataPtr();
  uint8_t size = arqLLI_getSize();

  //전송 받은 데이터의 정보를 출력해줍니다.
  pc.printf("\n -------------------------------------------------
  \nRCVD from %i : %s (length:%i, seq:%i)\n
  -------------------------------------------------\n",
  srcId, arqMsg_getWord(dataPtr), size, arqMsg_getSeq(dataPtr));
  //정상적으로 패킷을 전송 받았으니, 이를 src 쪽에 알려주기 위해 데이터를 인코딩하여
  //ACK를 보내줍니다.
  //ACK transmission
  arqMsg_encodeAck(arqAck, arqMsg_getSeq(dataPtr));
  arqLLI_sendData(arqAck, ARQMSG_ACKSIZE, srcId);
  
  //메인 상태를 TX으로 변경, flag_needPrint 변수의 값 변경, 이벤트를 초기화
  main_state = MAINSTATE_TX; //goto TX state
  flag_needPrint = 1;

  arqEvent_clearEventFlag(arqEvent_dataRcvd);
}
```
RX가 TX의 두 번째 전송 후에 TX와 연결되었습니다. 따라서 TX의 세 번째 retransmission으로 데이터를 수신 받게 됩니다. 메인이 IDLE인 상태에서 데이터 수신 이벤트를 처리한 후 동작을 종료합니다.

- - src node의 경우
```cpp
case MAINSTATE_ACK: //ACK state description
  //ACK를 드디어 받았다!
  if (arqEvent_checkEventFlag(arqEvent_ackRcvd)) //data TX finished
  { //수신 쪽과 송신 쪽의 ACK의 시퀀스가 동일한지 확인해 보는 작업
    uint8_t* dataPtr = arqLLI_getRcvdDataPtr();
    if ( arqMsg_getSeq(arqPdu) == arqMsg_getSeq(dataPtr) )
    { //ACK를 성공적으로 수신하였음을 출력, 타이머 종료, 메인 상태를 IDLE로 변경
    pc.printf("ACK is correctly received! \n");
    arqTimer_stopTimer();
    main_state = MAINSTATE_IDLE;
    }
    else
    { //아니라면 잘못된 ACK를 수신받았다고 알린다.
      pc.printf("ACK seq number is weird! (expected : %i, received : %i\n", arqMsg_getSeq(arqPdu),arqMsg_getSeq(dataPtr));
    }
```
패킷을 정상적으로 전송하여서 ACK를 수신 받았다는 것을 사용자에게 보여줍니다. 메인 상태를 IDLE로 바꾸어 일련의 동작을 마칩니다. 모든 동작이 끝났습니다!

<br>
<br>

3. Flowchart
- 전체 Flowchart
![pic](https://imgur.com/p31jT88.jpg)

- 송신 src 쪽 Flowchart
![pic](https://imgur.com/zED2RQC.jpg)
