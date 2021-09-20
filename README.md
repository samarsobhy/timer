 // LIB Layer
#include "STD_TYPES.h"
#include "BIT_MATH.h"

// MCAL
#include "TMR0_interface.h"
#include "TMR0_private.h"
#include "TMR0_config.h"

/*
// ( DESIRED_TIME < Time_OVF )
#define TMR0_u8_TMR0_2PRELOAD_VALUE 	TMR0_u16_RESOLUTION - DESIRED_TIME

static u32 Time_OVF ,TMR0_NB_OVERFLOW , TMR0_u32Preload_OVF;
void TMR0_Calculate (void)
{
	static const u16 Prescaler_Value[6]={0,1,8,64,256,1024};
//( DESIRED_TIME == Time_OVF )
   static  f32 Tick_Time ;
   Tick_Time =  (Prescaler_Value[TMR0_u8_PRESCALER] )/ F_CPU ;
   Time_OVF  =  Tick_Time * TMR0_u16_RESOLUTION ;

//( DESIRED_TIME > Time_OVF )
 TMR0_NB_OVERFLOW = DESIRED_TIME / Time_OVF ;
static  f32 TMR0_f32Tick_OVF ;
TMR0_f32Tick_OVF = (((f32)DESIRED_TIME / Time_OVF) - TMR0_NB_OVERFLOW);
 TMR0_u32Preload_OVF = TMR0_f32Tick_OVF * TMR0_u16_RESOLUTION;
}

*/

// Global Pointers to Function
static void(*TMR_PfTimer0_OVFCallBack)(void) = NULL;
static void(*TMR_PfTimer0_CTCCallBack)(void) = NULL;

void TMR0_voidTMR0Init (void)
{
	////////////////Select Mode//////////////////////////////////////
#if 	( TMR0_u8_MODE == TMR0_u8_NORMAL )
	CLEAR_BIT (TMR0_u8_TMR0_TCCR0_REG,TMR0_u8_TMR0_WGM00_BIT);
	CLEAR_BIT (TMR0_u8_TMR0_TCCR0_REG,TMR0_u8_TMR0_WGM01_BIT);
#elif	( TMR0_u8_MODE == TMR0_u8_CTC )
	CLEAR_BIT (TMR0_u8_TMR0_TCCR0_REG,TMR0_u8_TMR0_WGM00_BIT);
	  SET_BIT (TMR0_u8_TMR0_TCCR0_REG,TMR0_u8_TMR0_WGM01_BIT);
#endif
/////////////////Overflow Interrupt////////////////////////////////
#if 	( TMR0_u8_TMR0_OVF_INTERRUPT == TMR0_u8_TMR0_Enable_OVF_INTERRUPT )
	SET_BIT (TIMERS_u8_TMR_TIMSK_REG,TMR0_u8_TMR0_TOIE0_BIT);
#elif	( TMR0_u8_TMR0_OVF_INTERRUPT == TMR0_u8_TMR0_Disable_OVF_INTERRUPT )
	CLEAR_BIT (TIMERS_u8_TMR_TIMSK_REG,TMR0_u8_TMR0_TOIE0_BIT);
#endif
////////////////CTC Interrupt//////////////////////////////////////
#if ( TMR0_u8_TMR0_CTC_INTERRUPT == TMR0_u8_TMR0_Enable_CTC_INTERRUPT )
	SET_BIT (TIMERS_u8_TMR_TIMSK_REG,TMR0_u8_TMR0_OCIE0_BIT);
#elif	( TMR0_u8_TMR0_CTC_INTERRUPT == TMR0_u8_TMR0_Disable_CTC_INTERRUPT )
	CLEAR_BIT (TIMERS_u8_TMR_TIMSK_REG,TMR0_u8_TMR0_OCIE0_BIT);
#endif
}
u8 TMR0_u8TMR0Start (u32 Copy_u32StartValue)
{
	u8 Local_u8ErrorState = STD_TYPES_OK;
	switch (Copy_u32StartValue)
	{
		case TMR0_u8_TMR0_0 :
			TMR0_u8_TMR0_TCNT0_REG = TMR0_u8_TMR0_0 ;
		break;
		case TMR0_u16_TMR0_FromStop:
		break;
		case TMR0_u16_TMR0_OCR0:
			TMR0_u8_TMR0_OCR0_REG = TMR0_u32_TMR0_COMPARE_VALUE;
		break;
		case TMR0_u16_TMR0_From_PV :
			TMR0_u8_TMR0_TCNT0_REG = TMR0_u16_TMR0_PRELOAD_VALUE ;
		break;
		default:
		Local_u8ErrorState = STD_TYPES_NOK ;
	}
	// Select Prescaler Value 
		TMR0_u8_TMR0_TCCR0_REG |= TMR0_u8_TMR0_PRESCALER;
return Local_u8ErrorState;	
}
void TMR0_voidTMR0Stop (void)
{
	// Clear Prescaler Value 
	TMR0_u8_TMR0_TCCR0_REG |= TMR0_u8_TMR0_NO_PRESCALER ;
}
u32 TMR0_u32Read_Counts(void)
{
	u32 Local_u32REGValue=0;
	Local_u32REGValue=TMR0_u8_TMR0_TCNT0_REG;
	return Local_u32REGValue ;
}
u8 TMR0_u8TMR0_OVFSetCallBack (void(*Copy_PfCallBack)(void))
{
	u8 Local_u8ErrorState = STD_TYPES_OK;
	if (Copy_PfCallBack != NULL )
	{
		TMR_PfTimer0_OVFCallBack = Copy_PfCallBack ;
	}
	else
	{
		Local_u8ErrorState = STD_TYPES_NOK ;
	}
return Local_u8ErrorState;
}
u8 TMR0_u8TMR0_CTCSetCallBack (void(*Copy_PfCallBack)(void))
{
	u8 Local_u8ErrorState = STD_TYPES_OK;
	if (Copy_PfCallBack != NULL )
	{
		TMR_PfTimer0_CTCCallBack = Copy_PfCallBack ;
	}
	else
	{
		Local_u8ErrorState = STD_TYPES_NOK ;
	}
return Local_u8ErrorState;	
}
// ISR Function Prototype for TIMER0_CTC
void __vector_10(void)  __attribute__((signal));
void __vector_10(void)
{
#if( TMR0_u32_TMR0_DESIRED_TIME > TMR0_u32_TMR0_TIME_OVF )
	static u16 Local_u16Counter = 0 ;
	Local_u16Counter ++ ;
	if ( Local_u16Counter == TMR0_u32_TMR0_OVERFLOW_VALUE )
	{
		// Set Preload Value
		TMR0_u8_TMR0_TCNT0_REG = TMR0_u16_TMR0_PRELOAD_VALUE ;//TMR0_u8TMR0Start(TMR0_u16_TMR0_OCR0	);
		// CLEAR Counter
		Local_u16Counter = 0 ;
		// Set Call Back
		if ( TMR_PfTimer0_CTCCallBack != NULL)
		{
			TMR_PfTimer0_CTCCallBack();
		}
	}
#elif( TMR0_u32_TMR0_DESIRED_TIME < TMR0_u32_TMR0_TIME_OVF )
	// Set Preload Value
	TMR0_u8_TMR0_TCNT0_REG = TMR0_u16_TMR0_PRELOAD_VALUE ;//TMR0_u8TMR0Start(TMR0_u16_TMR0_OCR0	);

	// Set Call Back
		if ( TMR_PfTimer0_CTCCallBack != NULL)
		{
			TMR_PfTimer0_CTCCallBack();
		}

#elif ( TMR0_u32_TMR0_DESIRED_TIME == TMR0_u32_TMR0_TIME_OVF )
	// Set Call Back
	if ( TMR_PfTimer0_CTCCallBack != NULL)
	{
		TMR_PfTimer0_CTCCallBack();
	}
#endif
}


// ISR Function Prototype for TIMER0_OVF
void __vector_11(void)  __attribute__((signal));
void __vector_11(void)
{
#if( TMR0_u32_TMR0_DESIRED_TIME > TMR0_u32_TMR0_TIME_OVF )
	static u16 Local_u16Counter = 0 ;
	Local_u16Counter ++ ;
	if ( Local_u16Counter == TMR0_u32_TMR0_OVERFLOW_VALUE )
	{
		// Set Preload Value
		TMR0_u8_TMR0_TCNT0_REG = TMR0_u16_TMR0_PRELOAD_VALUE ;//TMR0_u8TMR0Start(TMR0_u16_TMR0_From_PV);
		// CLEAR Counter
		Local_u16Counter = 0 ;
		// Set Call Back
		if ( TMR_PfTimer0_OVFCallBack != NULL)
		{
			TMR_PfTimer0_OVFCallBack();
		}
	}
#elif( TMR0_u32_TMR0_DESIRED_TIME < TMR0_u32_TMR0_TIME_OVF )
	// Set Preload Value
	TMR0_u8_TMR0_TCNT0_REG = TMR0_u16_TMR0_PRELOAD_VALUE ;//TMR0_u8TMR0Start(TMR0_u16_TMR0_From_PV);

	// Set Call Back
		if ( TMR_PfTimer0_OVFCallBack != NULL)
		{
			TMR_PfTimer0_OVFCallBack();
		}

#elif ( TMR0_u32_TMR0_DESIRED_TIME == TMR0_u32_TMR0_TIME_OVF )
	// Set Call Back
	if ( TMR_PfTimer0_OVFCallBack != NULL)
	{
		TMR_PfTimer0_OVFCallBack();
	}
#endif
}
