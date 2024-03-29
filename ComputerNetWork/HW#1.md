# 컴퓨터네크워크 실습1 결과 보고서
> 201911011 컴퓨터과학과 1분반 정차미
<br>
1. 결과 캡처 (내용은 자율)

![img](https://imgur.com/d5TU7oS.png)
<br>
<br>
2. 동작 과정 설명

- **main.cpp**
```cpp
#include "mbed.h"
#include "string.h"

#include "ARQ_FSMevent.h"
#include "ARQ_msg.h"
#include "ARQ_timer.h"
#include "ARQ_LLinterface.h"
#include "ARQ_parameters.h"

//FSM state -------------------------------------------------
#define MAINSTATE_IDLE              0
#define MAINSTATE_TX                1

//GLOBAL variables (DO NOT TOUCH!) ------------------------------------------
//serial port interface
Serial pc(USBTX, USBRX);

//state variables
uint8_t main_state = MAINSTATE_IDLE; //protocol state

//source/destination ID
uint8_t endNode_ID=1;
uint8_t dest_ID=0;

//PDU context/size
uint8_t arqPdu[200];
uint8_t pduSize;

//SDU (input)
uint8_t originalWord[200];
uint8_t wordLen=0;



//ARQ parameters -------------------------------------------------------------
uint8_t seqNum = 0;     //ARQ sequence number
uint8_t retxCnt = 0;    //ARQ retransmission counter
uint8_t arqAck[5];      //ARQ ACK PDU
```

초기 설정을 위해 다른 파일을 include하고 변수를 지정합니다.

- - 메인 함수를 살펴봅니다. (main(void))
```cpp
//FSM operation implementation ------------------------------------------------
int main(void){
    uint8_t flag_needPrint=1;
    uint8_t prev_state = 0;

    //initialization
    pc.printf("------------------ ARQ protocol starts! --------------------------\n");
    arqEvent_clearAllEventFlag();
```
시작하기에 앞서 이벤트 Flag를 초기화합니다. (eventFlag = 0;)

```cpp
    //source & destination ID setting
    pc.printf(":: ID for this node : ");
    pc.scanf("%d", &endNode_ID);
    pc.printf(":: ID for the destination : ");
    pc.scanf("%d", &dest_ID);
    pc.getc();

    pc.printf("endnode : %i, dest : %i\n", endNode_ID, dest_ID);
```
TX와 RX의 번호를 입력해줍니다. 이때, src 노드의 ID 가 1이고, dest 노드의 ID가 2이라 하였으므로 endNode_ID에는 1, dest_ID에는 2가 들어갑니다. 사용자가 입력한 값을 출력해줍니다.

```cpp
    arqLLI_initLowLayer(endNode_ID);
    pc.attach(&arqMain_processInputWord, Serial::RxIrq);
```
arqLLI_initLowLayer 라는 함수를 호출합니다. 이 함수는 ARO_LLinterface.h 파일에 존재합니다. 시스템 콜인 attach 함수로 arqMain_processInputWord의 주소와 Rx를 넣어줍니다.

<br>
<br>

- ARQ_LLinterface
- - arqLLI_initLowLayer
```cpp
void arqLLI_initLowLayer(uint8_t srcId)
{
    phymac_init(srcId, arqLLI_dataCnfFunc, arqLLI_dataIndFunc);
}
```
phymac_init 함수는 PHYMAC_layer.h에 정의되어 있습니다. 또한 같은 ARQ_LLinterface 안에서 다른 함수를 호출합니다.

- - arqMain_processInputWord
```cpp
void arqMain_processInputWord(void)
{
    char c = pc.getc();
    if (main_state == MAINSTATE_IDLE &&
        !arqEvent_checkEventFlag(arqEvent_dataToSend))
    {
```
Main state와 MAINSTATE_IDLE 이 동일하고 (둘다 초기에 0으로 초기화, 프로토콜 상태가 같다는 것을 의미) 데이터를 보내야하는 이벤트를 통해 Flag를 체크했을 때 이 값이 0이 되면 조건문이 실행됩니다. (앞에 NOT이 붙었으므로)

- ARQ_FSMevent
```cpp
int arqEvent_checkEventFlag(arqEvent_e event)
{
    return (eventFlag & (0x01 << event));
}
```
event만큼 16비트로 1을 왼쪽으로 shift합니다. 이 값을 eventFlag와 AND 연산시켜 리턴해줍니다.

```cpp
        if (c == '\n' || c == '\r')
        {
            originalWord[wordLen++] = '\0';
            arqEvent_setEventFlag(arqEvent_dataToSend);
            pc.printf("word is ready! ::: %s\n", originalWord);
        }
```
입력받은 문자열이 ‘다음줄’ 혹은 ‘맨앞으로’였을 때 입력이 종료되었으므로 조건문이 실행됩니다. 문자열 가장 마지막에 NULL을 추가해줍니다. 입력 받은 문자열(originalWord)을 출력합니다.

```cpp
        else
        {
            originalWord[wordLen++] = c;
            if (wordLen >= ARQMSG_MAXDATASIZE-1)
            {
                originalWord[wordLen++] = '\0';
                arqEvent_setEventFlag(arqEvent_dataToSend);
                pc.printf("\n max reached! word forced to be ready :::: %s\n", originalWord);
            }
        }
    }
}
```
아닐 경우에는 문자열을 계속 입력받습니다. 문자열의 크기가 배열의 범위를 초과했을 경우 (오버 플로우 발생) 입력을 강제적으로 마친 후 결과값을 출력해줍니다.

---

- - 메인 함수의 반복문을 살펴봅니다.
```cpp
    while(1)
    {
        //debug message
        if (prev_state != main_state)
        {
            debug_if(DBGMSG_ARQ, "[ARQ] State transition from %i to %i\n", prev_state, main_state);
            prev_state = main_state;
        }

```
현재 상태가 메인 상태와 같지 않다면, 디버그일 경우 에러가 났음을 알리고 현재 상태를 메인 상태와 같도록 동기화시켜줍니다. 

```cpp
        //FSM should be implemented here! ---->>>>
        switch (main_state)
        {
            case MAINSTATE_IDLE: //IDLE state description
                
                if (arqEvent_checkEventFlag(arqEvent_dataRcvd)) //if data reception event happens
                {
                    //Retrieving data info.
                    uint8_t srcId = arqLLI_getSrcId();
                    uint8_t* dataPtr = arqLLI_getRcvdDataPtr();
                    uint8_t size = arqLLI_getSize();
```
IDLE 상태일 때 데이터를 수신 받으면, 정보를 각각 변수에 저장해줍니다. 

```cpp
                    pc.printf("\n -------------------------------------------------\nRCVD from %i : %s (length:%i, seq:%i)\n -------------------------------------------------\n", 
                                srcId, arqMsg_getWord(dataPtr), size, arqMsg_getSeq(dataPtr));
                    main_state = MAINSTATE_IDLE;
                    flag_needPrint = 1;

                    arqEvent_clearEventFlag(arqEvent_dataRcvd);
                }
```
입력 받은 정보를 출력해줍니다. 다른 노드의 ID(1 또는 2), 데이터의 내용(hello), 데이터의 사이즈, 송수신 순서를 차례대로 보여줍니다. 메인 상태를 업데이트한 뒤 Flag를 초기화해줍니다.

```cpp          
                else if (arqEvent_checkEventFlag(arqEvent_dataToSend)) //if data needs to be sent (keyboard input)
                {
                    //msg header setting
                    pduSize = arqMsg_encodeData(arqPdu, originalWord, seqNum, wordLen);
                    arqLLI_sendData(arqPdu, pduSize, dest_ID);
```
IDLE 상태일 때 데이터를 송신할 거라면, 메세지의 헤더를 설정해줍니다. PDU의 사이즈를 넣어주고 arqLLI_sendData 함수를 호출하는데, 이 함수는 "ARQ_LLinterface.h" 파일에 있습니다.
- - - arqLLI_sendData
```cpp
//TX function
void arqLLI_sendData(uint8_t* msg, uint8_t size, uint8_t dest)
{
    phymac_dataReq(msg, size, dest);
    txType = msg[ARQMSG_OFFSET_TYPE];
}

```
메세지의 내용과 사이즈, 목적지를 맥 레이어에 넣어주며 메세지를 통해 Tx 타입을 설정해줍니다.

```cpp

                    pc.printf("[MAIN] sending to %i (seq:%i)\n", dest_ID, (seqNum-1)%ARQMSSG_MAX_SEQNUM);

                    main_state = MAINSTATE_TX;
                    flag_needPrint = 1;

                    wordLen = 0;
                    arqEvent_clearEventFlag(arqEvent_dataToSend);
                }
```
사용자에게 전송 상태를 보여줍니다. 목적지 Rx의 ID와 송수신 순서를 출력합니다. 단어의 길이를 0으로 초기화시키고 나면 아래 과정은 메세지를 수신 받았을 때와 똑같습니다.

```cpp
                else if (flag_needPrint == 1)
                {
                    pc.printf("Give a word to send : ");
                    flag_needPrint = 0;
                }     

                break;
```
한 동작이 끝나고 나면 위의 문장을 출력한 후에 flag_needPrint를 다시 초기화시켜줍니다.
```cpp
            case MAINSTATE_TX: //IDLE state description

                if (arqEvent_checkEventFlag(arqEvent_dataTxDone)) //data TX finished
                {
                    main_state = MAINSTATE_IDLE;
                    arqEvent_clearEventFlag(arqEvent_dataTxDone);
                }

                break;

            default :
                break;
        }
    }
}
```
TX 상태일 때는 데이터 송수진이 종료된 것으로 확인하고 메인 상태와 Flag를 초기화시켜줍니다.
