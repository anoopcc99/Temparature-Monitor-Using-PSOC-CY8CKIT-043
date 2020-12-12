# Temparature-Monitor-Using-PSOC-CY8CKIT-043
Using SAR ADC and WDT in PSOC 4200M series Prototyping kit monitoring of Temperature.

![](https://github.com/anoopcc99/Temparature-Monitor-Using-PSOC-CY8CKIT-043/blob/main/images/1604468252809.jpg)

![](https://github.com/anoopcc99/Temparature-Monitor-Using-PSOC-CY8CKIT-043/blob/main/images/Figure1.jpg)

https://www.cypress.com/file/443811/download

![](https://github.com/anoopcc99/Temparature-Monitor-Using-PSOC-CY8CKIT-043/blob/main/images/ice_screenshot_20201104-111306.png)

## Components Used
* PSOC CY8CKIT-043
* LM35 Temparature Sensor

### LM35:
DataSheet:

![](https://github.com/anoopcc99/Temparature-Monitor-Using-PSOC-CY8CKIT-043/blob/main/images/ice_screenshot_20200928-165630.png)

### PSOC:

![](https://github.com/anoopcc99/Temparature-Monitor-Using-PSOC-CY8CKIT-043/blob/main/images/ice_screenshot_20201104-111800.png)

![](https://github.com/anoopcc99/Temparature-Monitor-Using-PSOC-CY8CKIT-043/blob/main/images/ice_screenshot_20201104-111353.png)

## Code:
```
#include <project.h>

#define CONVERT_TO_ASCII    	(0x30u)

#define CHANNEL_1           	(0u)

volatile uint8  dataReady  = 0u;
volatile uint32 limitFlag = 0u;

CY_ISR_PROTO(ADC1_ISR_Handler);


/* ADC sample ans convert */
static int16 ADC1_Convert(uint32 channel);
/* Process the data */
static void Data_Process(int16 mVolts);

static void SendDecimal(int16 txVal);

int main()
{
    int16 mVolts_ADC = 0u;
    Opamp_Start();
    ADC1_Start();
    UART_Start();

    ADC1_IRQ_StartEx(ADC1_ISR_Handler);
    
    CyGlobalIntEnable;      /* Enable global interrupts */
    
    /* Place your initialization/startup code here (e.g. MyInst_Start()) */
    
    for(;;)
    {
        /* Place your application code here */
        /* ADC conversion */
		mVolts_ADC = ADC1_Convert(CHANNEL_1);
		/* Process the voltage value */
		Data_Process(mVolts_ADC);
		
		/* Delay for UART Print completed */
		CyDelayUs(8000);
        CyDelay(1000);
    }
}

static void SendDecimal(int16 txVal)
{
	uint8 bitNum = 4;
	int16 val = 0;
	uint8 dataArray[bitNum];
	uint8 i = 0u;
	uint8 flag = 0;
	
	/* Convert txVal to ASCII */
	if(txVal < 0)
	{
		UART_UartPutChar('-');
		val = -txVal;
	}
	else
	{
		val = txVal;
	}
	
	for(i=0u; i<bitNum; i++)
	{
		dataArray[i] = val%10 + CONVERT_TO_ASCII;
		val /= 10;
	}
	
	/* Send Volts to UART */
	for(i= bitNum; i>0; i--)
	{
		/* if the MSB is 0, do not send it */
		if(flag == 0u)
		{
			if(dataArray[i-1] != (0+CONVERT_TO_ASCII))
			{
				flag = 1u;
			}
		}
		
		if(flag != 0u)
		{
			UART_UartPutChar(dataArray[i-1]);
		}	
	}
}

CY_ISR(ADC1_ISR_Handler)
{
    uint32 intr_status;

    /* Read interrupt status registers */
    intr_status = ADC1_SAR_INTR_MASKED_REG;
    /* Check for End of Scan interrupt */
    if((intr_status & ADC1_EOS_MASK) != 0u)
    {
        /* Read range interrupt status and raise the flag */
        limitFlag = ADC1_SAR_RANGE_INTR_MASKED_REG;
        /* Clear range detect status */
        ADC1_SAR_RANGE_INTR_REG = limitFlag;
        dataReady = 1u;
    }
    /* Clear handled interrupt */
    ADC1_SAR_INTR_REG = intr_status;
}

#define ADC_SAMPLE_CNT (10)
static int16 ADC1_Convert(uint32 channel)
{
	int16 mVolts = 0; 
	uint8 i = 0u;
	int32 ADCResultSum = 0;
	
	for(i = 0; i<ADC_SAMPLE_CNT; i++)
	{
	    /* Start ADC conversion */
	    ADC1_StartConvert();
		while(dataReady == 0u)
	    {
	        ; /* Wait for ADC conversion */
	    }
	    /* Stop ADC conversion */		
	    ADC1_StopConvert();
		dataReady = 0u;
		
		ADCResultSum += (int32)ADC1_GetResult16(channel);
	}

    /* Convert the ADC counts to mVolts. ADC is configured to be single channel */
	mVolts = ADC1_CountsTo_mVolts(channel, (int16)(ADCResultSum/ADC_SAMPLE_CNT));
	
	return mVolts;
}

static void Data_Process(int16 mVolts)
{
    int16 mVolts_ADC  = 0; 
	int32 TempC  = 0;
    //static int16 preVal = 0;
	static uint32 prelimitFlag = 0;
	
	mVolts_ADC = mVolts;
	TempC = ((float32)mVolts_ADC/10);
	
    /* Check ADC limit detection interrupt */
	if(prelimitFlag != limitFlag)
	{
		/* Display "Day" when input is below the low limit voltage, or display "Night" */
		UART_UartPutString((limitFlag > 0u) ? "Day" : "Night");
		UART_UartPutCRLF(0u);
		prelimitFlag = limitFlag;
	}
    
    /* If the result changes, send it to UART */
    //preVal != TempC
    if(1)
    {    
#if 1
		/* Send ADC Result Voltage to UART */
		UART_UartPutString("ADCRst:");
	    /* Send Volts to UART */
		SendDecimal(mVolts);
	    UART_UartPutString(" mV; ");
        UART_UartPutString("\n");
        UART_UartPutString("\r");
        UART_UartPutString("TempC:");
        SendDecimal(TempC);
        UART_UartPutString("*C");
        UART_UartPutString("\n");
        UART_UartPutString("\r");
#endif			
        //preVal = TempC;
    }
}
```
