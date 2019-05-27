# Заметки по поводу реализации адаптера для ClickHouse в Сокол Аналитика и не только


Python;
- Дравер - [ClickHouse Python Driver](https://github.com/mymarilyn/clickhouse-driver)
- Асинхронный сброс буфера поступивших сообщений - http://qaru.site/questions/381762/break-the-function-after-certain-time
```
import signal

class TimeoutException(Exception):   # Custom exception class
    pass

def timeout_handler(signum, frame):   # Custom signal handler
    raise TimeoutException

# Change the behavior of SIGALRM
signal.signal(signal.SIGALRM, timeout_handler)

for i in range(3):
    # Start the timer. Once 5 seconds are over, a SIGALRM signal is sent.
    signal.alarm(5)    
    # This try/except loop ensures that 
    #   you'll catch TimeoutException when it sent.
    try:
        A(i) # Whatever your function that might hang
    except TimeoutException:
        continue # continue the for loop if function A takes more than 5 second
    else:
        # Reset the alarm
        signal.alarm(0)
```
