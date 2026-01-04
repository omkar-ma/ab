# ab


#define F_CPU 16000000UL
#include <avr/io.h>
#include <avr/interrupt.h>

void UART_init(void)
{
    UCSR0A = 0;                 // Normal speed
    UBRR0H = 0;
    UBRR0L = 103;               // 9600 baud

    UCSR0B = (1 << RXEN0) | (1 << TXEN0) | (1 << RXCIE0);
    UCSR0C = (1 << UCSZ01) | (1 << UCSZ00);
}

void UART_transmit(char data)
{
    while (!(UCSR0A & (1 << UDRE0)));
    UDR0 = data;
}

ISR(USART_RX_vect)
{
    volatile char ch = UDR0;
    UART_transmit(ch);
}

int main(void)
{
    UART_init();
    sei();

    while (1)
    {
        // CPU free
    }
}












#define F_CPU 16000000UL

#include <avr/io.h>
#include <avr/interrupt.h>
#include <util/delay.h>

void UART_init(void)
{
    UCSR0A = 0;                 // Normal speed
    UBRR0H = 0;
    UBRR0L = 103;               // 9600 baud @ 16 MHz

    UCSR0B = (1 << RXEN0) | (1 << TXEN0) | (1 << RXCIE0);
    UCSR0C = (1 << UCSZ01) | (1 << UCSZ00);   // 8N1
}

void UART_transmit(char data)
{
    while (!(UCSR0A & (1 << UDRE0)));
    UDR0 = data;

    _delay_ms(2);   // ✅ small delay (2 ms)
}

ISR(USART_RX_vect)
{
    char ch = UDR0;       // Read received data
    UART_transmit(ch);    // Echo back
}

int main(void)
{
    UART_init();
    sei();                // Enable global interrupts

    while (1)
    {
        // CPU is free
    }
}
