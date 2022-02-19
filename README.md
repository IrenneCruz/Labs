; Archivo:	postlab4.s
; Dispositivo:	PIC16F887
; Autor:	Irenne Cruz
; Compilador:	pic-as (v2. 30), MPLABX V5. 40
;    
; Porgrama: Interrupción usando RB0 y RB7 para incrementar y decrementar contador y contador con tmr0
; Hardware: LEDs en puerto E y puerto C y pushbuttons en puerto B y display en puerto A y D
    
; Creado: 19 feb, 2022
; Última modificación: 19 feb, 2022

    
PROCESSOR 16F887
#include <xc.inc>

;configuracion de word 1
 CONFIG FOSC = INTRC_NOCLKOUT	; oscilador interno sin salidas
 CONFIG WDTE = OFF  ; reinicio repititivo del pic apagado    
 CONFIG PWRTE = OFF  ; espera de 72ms al inciar habilitada
 CONFIG MCLRE = OFF ; pin MCLRE se usa como In/Out
 CONFIG CP = OFF    ; sin protección de código
 CONFIG CPD = OFF   ; sin protección de datos

 CONFIG BOREN = OFF ; sin reinicio cuando el voltaje de alimentacion baja de 4V
 CONFIG IESO = OFF  ; reinicio sin cambio de reloj de interno a externo
 CONFIG FCMEN = OFF ; cambio de reloj externo a interno si hay fallo
 CONFIG LVP = OFF   ; programacion en bajo voltaje permitida
 
;configuracion word 2
 CONFIG WRT = OFF	; poteccion autoescritura por el programa apagada
 CONFIG BOR4V = BOR40V	; reinicio abajo de 4V, (BOR21V=2.1v)
      
 UP EQU 0
 DOWN EQU 7 

 restart_tmr0 macro
     banksel PORTC
     movlw 171		;estableciendo valor inicial para el tmr
     movwf TMR0
     bcf T0IF
     endm 
    
 restart_tmr02 macro
     banksel PORTA
     movlw 60		;estableciendo valor inicial para el tmr
     movwf TMR0
     bcf T0IF
     endm      
    
    
; variables
 PSECT udata_bank0 ;common memory
     cont1: DS 2 ;2 byte
     cont: DS 2 ;2 byte
    
 PSECT udata_shr ;common memory
     W_TEMP: DS 1 ;1 byte
     STATUS_TEMP: DS 1    
    
 PSECT resVect, class = CODE, abs, delta=2    
;------------vector reset------------  
 ORG 00h
 
 resetVec:
     PAGESEL main
     goto main

 PSECT intVect, class = CODE, abs, delta=2     
;----------vector de interrupción----------- 
 ORG 04h   
 
 push:
     movwf W_TEMP
     swapf STATUS, W
     movwf STATUS_TEMP
    
 isr:
     btfsc T0IF
     btfsc RBIF
     call config_iocb
     call int_t0
     call int_t02
    
 pop:    
     swapf STATUS_TEMP, W
     movwf STATUS
     swapf W_TEMP, F
     swapf W_TEMP, W
     retfie
    
 config_iocb:
     banksel PORTE
     btfss PORTB, UP
     incf PORTE
     btfsc PORTE, 4
     clrf PORTE
     btfss PORTB, DOWN
     decf PORTE
     btfsc PORTE, 7  ; detener la cuenta si debe encenderse el puerto 7
     bcf PORTE, 4    ; apagar del puerto 4 en adelante
     bcf PORTE, 5 
     bcf PORTE, 6
     bcf PORTE, 7
     bcf RBIF
     return
    
 int_t0:    
     restart_tmr0
     incf cont1
     movf cont1, W
     sublw 20
     btfss ZERO	    ; status, 2
     goto ret_t0
     clrf cont1
     incf PORTC, F
     btfsc PORTC, 4
     clrf PORTC
     
 ret_t0:
     return
    
 int_t02:    
     restart_tmr02
     incf cont
     movf cont, W
     sublw 199
     btfss ZERO	    ; status, 2
     goto ret_t02
     clrf cont
     incf PORTA, F
     movf PORTA, W
     call tabla
     movwf PORTA
     
 ret_t02:
     return    
    
    
 PSECT code, delta = 2, abs
;------------configuracion del uC------------
 ORG 100h 
 
 tabla: 
     banksel PORTC
     clrf PCLATH ; clear al pclath
     bsf PCLATH, 0 ;pclath = 01
     andlw 0x0f
     addwf PCL ;PC = PCLATH + PCL
     retlw 00111111B ;0
     retlw 00000110B ;1
     retlw 01011011B ;2
     retlw 01001111B ;3
     retlw 01100110B ;4
     retlw 01101101B ;5
     retlw 01111101B ;6
     retlw 00000111B ;7
     retlw 01111111B ;8
     retlw 01101111B ;9    
   
 main:
     call config_io	; llamada a la subrutina de ins y out
     call config_reloj  ; configuración del reloj
     call config_ioc	; subrutina de interrupt on change
     call config_t0int
     call config_intrpt
     banksel PORTA
     clrf PORTA		;inicio en 0
     banksel PORTC 
     clrf PORTC		;inicio en 0
     banksel PORTD
     clrf PORTD
     banksel PORTE
     clrf PORTE

;------------loop principal------------
 loop:
     banksel PORTC
     movf PORTC, W
     call tabla
     movwf PORTD
     goto loop	; loop forever

 config_reloj:	; configuración del reloj con los registros del OSCCON para definir el oscilador
     banksel OSCCON
     bsf IRCF2	    ;OSCCON, 6 / bit 2 del IRCF en set
     bsf IRCF1	    ;OSCCON, 6 / bit 1 del IRCF en set 
     bcf IRCF0	    ;OSCCON, 6 / bit 0 del IRCF en clear y oscilador de 4MHz configurado (110)
     bsf SCS	    ;reloj interno en set
     return

 config_ioc:
     banksel TRISA
     bsf IOCB, UP 
     bsf IOCB, DOWN ;habilitan las interrupciones en cambio para B
     banksel PORTA
     movf PORTB, W  ;al leer termina la condicion mismatch
     bcf RBIF
     return
    
 config_t0int:    
     banksel TRISC
     bcf T0CS	;este selecciona el reloj al que el timer estara ligado (0 = interno)
     bcf PSA	;prescaler asignado al tmr0 (1 = watchdog timer)
     bsf PS2	;bits para el rango del prescaler 
     bsf PS1
     bsf PS0	;presclar asignado en 256 (111)
     restart_tmr0
     return    
    
 config_io: ;subrutina entradas y salidas
     bsf STATUS, 5 ;BANK 11
     bsf STATUS, 6 
     clrf ANSEL ; pines digitales
     clrf ANSELH 
    
     bsf STATUS,5 ; BANK 01
     bcf STATUS, 6
     clrf TRISA ; port a como out
     clrf TRISC ; port c como out
     clrf TRISD ; port d como out
     clrf TRISE ; port e como out
     bsf TRISB, UP 
     bsf TRISB, DOWN	;port b como in
     bcf OPTION_REG, 7	;habilitando los pull ups del puert B
     bsf WPUB, UP 
     bsf WPUB, DOWN	;UP y DOWN como entradas
    
    
     bcf STATUS, 5  ; BANK 00
     bcf STATUS, 6
    
 config_intrpt:
     bsf GIE	    ;INTCON
     bsf T0IE
     bsf RBIE	    ;enable de ambas interrupciones
     bcf T0IF
     bcf RBIF	    ; banderas
     return       
    
END
