
#include <avr/io.h>
#include <avr/interrupt.h>

void setup()
{
    // Enable global interrupt
    sei();

    // ---------------- ADC CONFIGURATION ----------------

    // Set reference to AREF (NO internal reference), select ADC0 initially
    ADMUX &= ~(1 << REFS0);
    ADMUX &= ~(1 << REFS1);

    // Right adjust ADC result
    ADMUX &= ~(1 << ADLAR);

    // Enable ADC, ADC Interrupt, Auto Trigger
    // Prescaler = 16
    ADCSRA |= (1 << ADEN) | (1 << ADIE) | (1 << ADATE) | (1 << ADPS2);
    ADCSRA &= ~((1 << ADPS1) | (1 << ADPS0));

    // ADC clock = 16 MHz / 16 = 1 MHz

    // Start ADC conversion
    ADCSRA |= (1 << ADSC);

    // Select ADC0 channel
    ADMUX &= ~(1 << MUX0);
    ADMUX &= ~(1 << MUX1);
    ADMUX &= ~(1 << MUX2);
    ADMUX &= ~(1 << MUX3);
}

void loop()
{
    // Main loop empty
    // ADC handled by interrupt
}

// ---------------- ADC INTERRUPT ----------------
ISR(ADC_vect)
{
    OCR1A = ADC;   // ADC value: 0–1023
}

