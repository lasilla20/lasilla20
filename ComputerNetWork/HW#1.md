
```

41.	//application event handler : generating SDU from keyboard input
42.	void arqMain_processInputWord(void)
43.	{
44.	    char c = pc.getc();
45.	    문자열을 입력받습니다.
46.	    if (main_state == MAINSTATE_IDLE &&
47.	        !arqEvent_checkEventFlag(arqEvent_dataToSend))
48.	Main state와 MAINSTATE_IDLE 이 동일하고, (둘다 초기에 0으로 초기화)
49.	데이터를 보내야하는 이벤트를 통해 Flag를 체크했을 때 이 값이 0이 되면 조건문이 실행된다. (앞에 NOT이 붙었으므로)
50.	    {
51.	        if (c == '\n' || c == '\r')
52.	입력받은 문자열이 ‘다음줄’ 혹은 ‘맨앞으로’였을 때 입력이 종료되었으므로
53.	조건문이 실행된다. 문자열 가장 마지막에 NULL을 추가해준다.
54.	        {
55.	            originalWord[wordLen++] = '\0';
56.	            arqEvent_setEventFlag(arqEvent_dataToSend);
57.	            pc.printf("word is ready! ::: %s\n", originalWord);
58.	입력 받은 문자열(originalWord)을 출력한다.
59.	        }
60.	        else
아닐 경우에는 문자열을 계속 입력받는다.
61.	        {
62.	            originalWord[wordLen++] = c;
63.	            if (wordLen >= ARQMSG_MAXDATASIZE-1)
문자열의 크기가 배열의 범위를 초과했을 경우 (오버 플로우 발생) 입력을 강제적으로 마친 후 결과값을 출력해준다.
64.	            {
65.	                originalWord[wordLen++] = '\0';
66.	                arqEvent_setEventFlag(arqEvent_dataToSend);
67.	                pc.printf("\n max reached! word forced to be ready :::: %s\n", originalWord);
68.	            }
69.	        }
70.	    }
71.	}
72.	
73.	
74.	
75.	
76.	//FSM operation implementation ------------------------------------------------
77.	int main(void){
78.	    uint8_t flag_needPrint=1;
79.	    uint8_t prev_state = 0;
80.	
81.	    //initialization
82.	    pc.printf("------------------ ARQ protocol starts! --------------------------\n");
83.	    arqEvent_clearAllEventFlag();
시작하기에 앞서 이벤트 Flag를 초기화합니다.
84.	   
85.	    //source & destination ID setting
86.	    pc.printf(":: ID for this node : ");
87.	    pc.scanf("%d", &endNode_ID);
88.	    pc.printf(":: ID for the destination : ");
89.	    pc.scanf("%d", &dest_ID);
90.	    pc.getc();
91.	endNode와 목적지의 값을 입력해줍니다. 이때, src 노드의 ID 가 1이고, dest 노드의 ID가 2이라 하였으므로 endNode_ID에는 1, dest_ID에는 2가 들어갑니다.
92.	    pc.printf("endnode : %i, dest : %i\n", endNode_ID, dest_ID);
93.	    arqLLI_initLowLayer(endNode_ID);
94.	입력값을 보여주고 “arqLLI_initLowLayer” 라는 함수를 호출합니다.
95.	    pc.attach(&arqMain_processInputWord, Serial::RxIrq);
96.	    arqMain_processInputWord의 주소와 시리얼을 넣어 attch를 호출합니다.
97.	
98.	
99.	
100.	
101.	    while(1)
102.	    {
103.	        //debug message
104.	        if (prev_state != main_state)
105.	        {
106.	            debug_if(DBGMSG_ARQ, "[ARQ] State transition from %i to %i\n", prev_state, main_state);
107.	            prev_state = main_state;
108.	        }
109.	
110.	
111.	        //FSM should be implemented here! ---->>>>
112.	        switch (main_state)
113.	        {
114.	            case MAINSTATE_IDLE: //IDLE state description
115.	                
116.	                if (arqEvent_checkEventFlag(arqEvent_dataRcvd)) //if data reception event happens
117.	                {
118.	                    //Retrieving data info.
119.	                    uint8_t srcId = arqLLI_getSrcId();
120.	                    uint8_t* dataPtr = arqLLI_getRcvdDataPtr();
121.	                    uint8_t size = arqLLI_getSize();
122.	
123.	                    pc.printf("\n -------------------------------------------------\nRCVD from %i : %s (length:%i, seq:%i)\n -------------------------------------------------\n", 
124.	                                srcId, arqMsg_getWord(dataPtr), size, arqMsg_getSeq(dataPtr));
125.	
126.	                    main_state = MAINSTATE_IDLE;
127.	                    flag_needPrint = 1;
128.	
129.	                    arqEvent_clearEventFlag(arqEvent_dataRcvd);
130.	                }
131.	                else if (arqEvent_checkEventFlag(arqEvent_dataToSend)) //if data needs to be sent (keyboard input)
132.	                {
133.	                    //msg header setting
134.	                    pduSize = arqMsg_encodeData(arqPdu, originalWord, seqNum, wordLen);
135.	                    arqLLI_sendData(arqPdu, pduSize, dest_ID);
136.	
137.	                    pc.printf("[MAIN] sending to %i (seq:%i)\n", dest_ID, (seqNum-1)%ARQMSSG_MAX_SEQNUM);
138.	
139.	                    main_state = MAINSTATE_TX;
140.	                    flag_needPrint = 1;
141.	
142.	                    wordLen = 0;
143.	                    arqEvent_clearEventFlag(arqEvent_dataToSend);
144.	                }
145.	                else if (flag_needPrint == 1)
146.	                {
147.	                    pc.printf("Give a word to send : ");
148.	                    flag_needPrint = 0;
149.	                }     
150.	
151.	                break;
152.	
153.	            case MAINSTATE_TX: //IDLE state description
154.	
155.	                if (arqEvent_checkEventFlag(arqEvent_dataTxDone)) //data TX finished
156.	                {
157.	                    main_state = MAINSTATE_IDLE;
158.	                    arqEvent_clearEventFlag(arqEvent_dataTxDone);
159.	                }
160.	
161.	                break;
162.	
163.	            default :
164.	                break;
165.	        }
166.	    }
167.	}
```
z코멘트
