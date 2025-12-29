# Ревью проекта 'Обмен валюты'
Исходный код - https://github.com/Ciltonn/CurrencyExchangeRate
# Понравилось:
- код запускается локально
- деление приложения на слои
- наличие интерфейсов в слое DAO
- валидация кода валюты с использованием стандарта ISO
- централизованная обработка пользовательских исключений
- наличие утилитных классов
# Надо доработать:
- надо удалить каталог .idea 
- заполнить ReadMe.md
- В pom.xml указана java 11. Желательно использовать более свежую, хотя бы java 17. Это позволит использовать современные возможности языка, такие как Record и текстовые блоки.
  
## Сервлеты
- пакет с сервлетами лучше все же назвать servlets.

Создание объектов одновременно с их объявлением как полей класса неправильно:
```
@WebServlet("/currencies")
public class CurrenciesServlet extends HttpServlet {

    private final CurrencyDaoImpl currencyDaoImpl = new CurrencyDaoImpl();
    private final ObjectMapper objectMapper = new ObjectMapper();
    ...
```
Вместо непосредственного создания надо выполнить внедрение объектов. Для этого сначала
поместим используемые объекты в контекст:
```
@WebListener
public class AppContextListener implements ServletContextListener {

    @Override
    public void contextInitialized(ServletContextEvent sce) {
        ObjectMapper objectMapper = new ObjectMapper();
        CurrencyDao currencyDao = new CurrencyDaoImpl();
        ExchangeRatesDao exchangeRates = new ExchangeRatesImpl();
        ExchangeRateService exchangeRateService = new ExchangeRateService(exchangeRates);
        CurrencyService currencyService = new CurrencyService(currencyDao);
        ServletContext context = sce.getServletContext();
        context.setAttribute("objectMapper", objectMapper);
        context.setAttribute("currencyDaoImpl", currencyDaoImpl);
        context.setAttribute("exchangeRateService", exchangeRateService);
        context.setAttribute("exchangeRatesImpl", exchangeRatesImpl);
        context.setAttribute("currencyService", currencyService);
    }
}
```
а затем внедрим их в сервлет, используя метод init() :
```
@WebServlet("/currencies")
public class CurrenciesServlet extends HttpServlet {

    private CurrencyService currencyService;
    private ObjectMapper objectMapper;

    @Override
    public void init() throws ServletException {
        super.init();
        ServletContext context = getServletContext();
        this.currencyService = (CurrencyService) context.getAttribute("currencyService");
        this.objectMapper = (ObjectMapper) context.getAttribute("objectMapper");
    }
    ...
```
Сервлет - это транспортный слой между фронтендом и бэкендом, его задача - извлечь нужные поля из HttpServletRequest request,
 отдать их на обработку в сервис бэкенда, а затем результат поместить в HttpServletResponse response. Поэтому убираем из текущей реализации
лишние операции:
```
    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws IOException {

        response.setContentType("application/json");
        response.setCharacterEncoding("UTF-8");

        List<Currency> currencies = currencyDaoImpl.findAll();

        List<CurrencyResponse> currencyResponse = currencies.stream()
                .map(MapperUtil::convertToDto)
                .collect(Collectors.toList());

        response.setStatus(HttpServletResponse.SC_OK);
        objectMapper.writeValue(response.getWriter(), currencyResponse);
    }
```
- изменение заголовков
```
        ...
        response.setContentType("application/json");
        response.setCharacterEncoding("UTF-8");
        ...
```
убираем из всех методов во всех сервлетах и переносим их в отдельный фильтр:
```
import jakarta.servlet.*;
import jakarta.servlet.annotation.WebFilter;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;

@WebFilter(urlPatterns = {"/currencies", "/currency/*", "/exchangeRate/*", "/exchangeRates" /* , "/exchange"*/})
public class JsonResponseFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        HttpServletResponse httpResp = (HttpServletResponse) response;
        httpResp.setContentType("application/json");
        httpResp.setCharacterEncoding("UTF-8");
        chain.doFilter(request, response);
    }
}
```
- код, в котором производится преобразование ответа репозитория
```
        ...
        List<Currency> currencies = currencyDaoImpl.findAll();
        List<CurrencyResponse> currencyResponse = currencies.stream()
                .map(MapperUtil::convertToDto)
                .collect(Collectors.toList());
        ...        
```
 убираем из сервлета и переносим в сервис:
```
    public class CurrencyService {

    private final CurrencyDao currencyDaoImpl;
    
        public CurrencyService(CurrencyDao currencyDaoImpl) {
            this.currencyDaoImpl = currencyDaoImpl;
        }
    
        public List<CurrencyRespDto> findAll() {
            List<Currency> currencies = currencyDaoImpl.findAll();
            return currencies.stream()
                    .map(c -> new CurrencyRespDto(c.getId(), c.getCode(), c.getName(), c.getSign()))
                    .toList();
        }
    
        public Currency save(CurrencyRequest currencyRequest) {
            ValidationUtil.validate(currencyRequest);
            return currencyDaoImpl.save(MapperUtil.convertToEntity(currencyRequest));
        }
    }
```
В итоге получаем такой сервлет:
```
@WebServlet("/currencies")
public class CurrenciesServlet extends HttpServlet {

    private CurrencyService currencyService;
    private ObjectMapper objectMapper;

    @Override
    public void init() throws ServletException {
        super.init();
        ServletContext context = getServletContext();
        this.currencyService = (CurrencyService) context.getAttribute("currencyService");
        this.objectMapper = (ObjectMapper) context.getAttribute("objectMapper");
    }

    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws IOException {
        List<CurrencyRespDto> currencies = currencyService.findAll();
        response.setStatus(HttpServletResponse.SC_OK);
        objectMapper.writeValue(response.getWriter(), currencies);
    }

    @Override
    protected void doPost(HttpServletRequest request, HttpServletResponse response) throws IOException {
        CurrencyRequest currencyRequest = new CurrencyRequest(
                request.getParameter("code"),
                request.getParameter("name"),
                request.getParameter("sign"));
        Currency currency = currencyService.save(currencyRequest);
        response.setStatus(HttpServletResponse.SC_CREATED);
        objectMapper.writeValue(response.getWriter(), MapperUtil.convertToDto(currency));
    }
}
```
Аналогичные по смыслу изменения надо выполнить во всех сервлетах. 
Валидацию входных параметров надо перенести в утилитный класс.
Этих операций  
```
        ...
        if((path == null) || (path.length() != 6)) {
            throw new InvalidParameterException("Invalid currency codes in path");
        }
        
        String baseCurrencyCode = path.substring(0, 3);
        String targetCurrencyCode = path.substring(3, 6);
        
        ValidationUtil.validateCurrencyCode(baseCurrencyCode);
        ValidationUtil.validateCurrencyCode(targetCurrencyCode);
        ...
```        
и этих
```
        ...
        if (parameter == null) {
            throw new InvalidParameterException("Missing parameter rate");
        }
        
        String rate = parameter.replace("rate=", "");
        
        if (rate.isBlank()) {
            throw new InvalidParameterException("Missing parameter rate");
        }
        
        BigDecimal rateBigDecimal;
            try {
                rateBigDecimal = new BigDecimal(rate);
            } catch (NumberFormatException e) {
                throw new InvalidParameterException("Invalid rate format");
            }
        ...    
```
в методах контроллеров быть не должно.

## Репозитории
При написании многострочных запросов лучше использовать текстовые блоки
```
"""
здесь 
  располагается 
    многострочный
запрос
"""
```

## DTO пакет
Вместо классов, используемых в качестве моделей для request и response
```
@Data
@NoArgsConstructor
@AllArgsConstructor

public class CurrencyRequest {

    private String code;
    private String name;
    private String sign;
}
```
лучше использовать Record:
```
public record CurrencyReqDto(String code, String name, String sign) {
}
```
Они неизменяемые, потокобезопасные, сразу содержат методы доступа и допускают использование аннотаций валидации прямо в конструкторе.

# Фильтры
Фильтр ExceptionFilter лучше перенести в этот пакет, к фильтрам.

Фильтр CorsFilter не требуется в текущей реализации, потому что и фронт, и бэк расположены на одном
хосте.

# Code Style
Требует упорядочивания использование пустых строк.
Методы должны отделяться друг от друга максимум одной строкой.
Внутри методов пустые строки не допускаются. Использование пустых строк внутри метода призвано визуально разделить функционал одной части метода от другой.
Это значит, что возникновение такой необходимости свидетельствует о том, что метод выполняет более чем одну задачу и должен быть разделен на несколько частей. 

# Итог
Проект легко читается, обладает хорошим неймингом полей и методов.
Необходимые доработки проекта, указанные в ревью, не должны занять много времени.



