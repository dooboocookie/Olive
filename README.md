# Olive Zero

- Role: Back-end Dev, JSP/Servlet, Java, Oracle, 팀장
- 소요기간: 2022.06 - 2022.07 (2주)
- 팀원 수: 5명

# 💡 프로젝트 기획

- 쇼핑몰 구현 웹 프로젝트
- 국내 1위 드러그 스토어인 `올리브영`을 벤치마킹 하여 쇼핑몰 구현하는 프로젝트
- [요구분석서 보기](https://github.com/dooboocookie/Olive/blob/main/ect/requirements_analysis/requirements_analysis.md)

---

# 🛠 개발 내용

## Tech Stack

- JAVA 8
- JSP / Servlet
- Oracle
- JDBC

## API

![KakaoTalk_Photo_2022-07-11-08-08-13-1.png](/ect/Olive%20Zero%201d9513c348cf45d7a9f379a247ec399c/KakaoTalk_Photo_2022-07-11-08-08-13-1.png)

## 주요 기능

### 프론트 컨트롤러

- properties에 요청 URL에 따른 핸들러 매핑 하여 서블릿의  init() 에서 핸들러 객체 생성하여 HashMap에  저장

```java
private Map<String, CommandHandler> commandHandlerMap = new HashMap<String, CommandHandler>();

public void init() throws ServletException {
	String path = this.getInitParameter("path");
	String realPath = this.getServletContext().getRealPath(path); 
	System.out.println(realPath);
	Properties prop = new Properties();
	try(FileReader fr = new FileReader(realPath)) {
		prop.load(fr);			
	} catch (Exception e) {
		throw new ServletException();
	}
	Iterator<Object> ir = prop.keySet().iterator(); 
	while (ir.hasNext()) {
		String url = (String) ir.next(); // 요청URL
		String commandHandlerFullName = prop.getProperty(url); // 키 값으로 프로퍼티를 가져옴
		// 객체 생성 ***
		try {
			Class<?> handlerClass = Class.forName(commandHandlerFullName); 
			CommandHandler commandHandler = (CommandHandler) handlerClass.newInstance();
			System.out.println(commandHandler);
			this.commandHandlerMap.put(url, commandHandler);
		} catch (ClassNotFoundException e) {
			e.printStackTrace();
		} catch (InstantiationException e) {
			e.printStackTrace();
		} catch (IllegalAccessException e) {
			e.printStackTrace();
		} // try
	} // while
} // init
```

- doGet() 과 doPost() 실행 시 요청URL을 키 값으로 갖는 핸들러 객체 호출하여 process()메소드 호출

```java
protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
	String requestURI = request.getRequestURI(); // contextPath부터 출력
	String contextPath = request.getContextPath();
	System.out.println(contextPath);
	
	if(requestURI.indexOf(contextPath) == 0) { 
		requestURI = requestURI.substring(contextPath.length()); 
	} // if	

	CommandHandler modelHandler = this.commandHandlerMap.get(requestURI); 

	String viewPage = null;
	try {
		viewPage = modelHandler.process(request, response); 
	} catch (Exception e) {
		e.printStackTrace();
	} // try
	if(viewPage != null) {
		RequestDispatcher dispatcher = request.getRequestDispatcher(viewPage);
		dispatcher.forward(request, response);
	} // if
} // doGet

protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
	doGet(request, response);
} // doPost
```

- 그 후 `핸들러(서비스 로직과 뷰를 연결)` > `서비스(하나의 트랜잭션에서 있어야할 비즈니스 로직 처리)` >  `DAO(DB에 쿼리문을 통하여 CRUD)` 로 이동하며 jsp로 포워딩하거나, 다른 요청으로 리다이렉트

### 카테고리

- 대분류, 중분류, 소분류, 상세분류 총 4개의 LEVEL로 되어있는 카테고리 테이블

![스크린샷 2022-11-30 오후 11.08.51.png](/ect/Olive%20Zero%201d9513c348cf45d7a9f379a247ec399c/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2022-11-30_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE_11.08.51.png)

- 대, 중, 소 카테고리를 가져오는 쿼리문 보기
    
    ```java
    public Map<CategoryDTO, Map<CategoryDTO, List<CategoryDTO>>> selectTop(Connection connection) {
          String sql1 = "SELECT * FROM category WHERE LEVEL = 1 START WITH ca_topcode IS NULL CONNECT BY PRIOR ca_code =ca_topcode ";
          String sql2 = "SELECT * FROM category WHERE ca_topcode = ? START WITH ca_topcode IS NULL CONNECT BY PRIOR ca_code =ca_topcode ";
          Map<CategoryDTO, Map<CategoryDTO, List<CategoryDTO>>> totMap = null;
          Map<CategoryDTO, List<CategoryDTO>> midMap = null;
          List<CategoryDTO> bottList = null;
          PreparedStatement pstmt1 = null;
          PreparedStatement pstmt2 = null;
          PreparedStatement pstmt3 = null;
          ResultSet rs1 = null;
          ResultSet rs2 = null;
          ResultSet rs3 = null;
    
          try {
              pstmt1 = connection.prepareStatement(sql1);
              rs1 = pstmt1.executeQuery();
              if (rs1.next()) {
                  totMap = new LinkedHashMap();
                  CategoryDTO topDTO = null;
    
                  do {
                      topDTO = new CategoryDTO();
                      topDTO.setCa_code(rs1.getString("ca_code"));
                      topDTO.setCa_name(rs1.getString("ca_name"));
                      topDTO.setCa_topcode(rs1.getString("ca_topcode"));
                      topDTO.setCa_level(rs1.getInt("ca_level"));
                      pstmt2 = connection.prepareStatement(sql2);
                      pstmt2.setString(1, topDTO.getCa_code());
                      rs2 = pstmt2.executeQuery();
                      if (rs2.next()) {
                          midMap = new LinkedHashMap();
                          CategoryDTO midDTO = null;
    
                          do {
                              midDTO = new CategoryDTO();
                              midDTO.setCa_code(rs2.getString("ca_code"));
                              midDTO.setCa_name(rs2.getString("ca_name"));
                              midDTO.setCa_topcode(rs2.getString("ca_topcode"));
                              midDTO.setCa_level(rs2.getInt("ca_level"));
                              pstmt3 = connection.prepareStatement(sql2);
                              pstmt3.setString(1, midDTO.getCa_code());
                              rs3 = pstmt3.executeQuery();
                              if (rs3.next()) {
                                  bottList = new ArrayList();
                                  CategoryDTO bottDTO = null;
    
                                  do {
                                      bottDTO = new CategoryDTO();
                                      bottDTO.setCa_code(rs3.getString("ca_code"));
                                      bottDTO.setCa_name(rs3.getString("ca_name"));
                                      bottDTO.setCa_topcode(rs3.getString("ca_topcode"));
                                      bottDTO.setCa_level(rs3.getInt("ca_level"));
                                      bottList.add(bottDTO);
                                  } while(rs3.next());
                              }
    
                              midMap.put(midDTO, bottList);
                          } while(rs2.next());
                      }
    
                      totMap.put(topDTO, midMap);
                  } while(rs1.next());
              }
    
              return totMap;
          } catch (SQLException var19) {
              throw new RuntimeException(var19);
          } finally {
              try {
                  pstmt1.close();
                  pstmt2.close();
                  pstmt3.close();
                  rs1.close();
                  rs2.close();
                  rs3.close();
              } catch (SQLException e) {
                  throw new RuntimeException(e);
              }
    
          }
      }
    ```
    
- 저 nav 정보는 모든 페이지에서 보는 정보이므로, 필터를 통하여 매 요청마다 해당 정보를 DB에서 가져옴

```java
@Override
public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
    CategoryMainService categoryMainService = CategoryMainService.getInstance();
    Map<CategoryDTO, Map<CategoryDTO, List<CategoryDTO>>> totMap = categoryMainService.selectTopCate();
    servletRequest.setAttribute("totMap", totMap);

    // 검색창 할인 TOP3 제품
    SearchTOP3ProductService searchTop3ProductService = SearchTOP3ProductService.getInstance();
    List<ProductBrandPriceDTO> searchTop3List = searchTop3ProductService.serachTop3ProductSelect();
    servletRequest.setAttribute("searchTop3List", searchTop3List);
    filterChain.doFilter(servletRequest, servletResponse); 
}
```

### 카테고리 별 상품 리스트

- 중분류 카테고리 선택 시 제품 리스트
    - 중분류의 자식으로 있는 소분류 카테고리 정보와 해당되는 제품들 리스트를 가져옴

![스크린샷 2022-11-30 오후 11.14.34.png](/ect/Olive%20Zero%201d9513c348cf45d7a9f379a247ec399c/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2022-11-30_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE_11.14.34.png)

- 소분류 카테고리 선택 시 제품 리스트
    - 소분류의 자식으로 있는 상세분류 카테도리 정보와 제품 리스트들과 그 제품들의 브랜드 리스트를 가져옴

![스크린샷 2022-11-30 오후 11.15.31.png](/ect/Olive%20Zero%201d9513c348cf45d7a9f379a247ec399c/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2022-11-30_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE_11.15.31.png)

- 카테고리 제품 리스트 불러오는 DAO 보기
    
    ```java
    @Override
    public List<ProductBrandPriceDTO> selectMCateTopVeiwProduct(Connection connection, String ca_code) {
        List<ProductBrandPriceDTO> list = null;
        String sql = "SELECT *  " +
                "FROM (  " +
                "    SELECT p.*, b.br_name, pi.prm_url , NVL(pr.prpri_price, 0) price, NVL(s.sa_rate, 0) sale_rate, NVL(pr.prpri_price, 0)*(1-NVL(s.sa_rate, 0)) realprice " +
                "    FROM product p  " +
                "        JOIN brand b ON p.br_code = b.br_code  " +
                "        LEFT OUTER JOIN productmimg pi ON p.pr_code = pi.pr_code  " +
                "        LEFT OUTER JOIN (SELECT * FROM prprice WHERE prpri_enddate IS NULL) pr ON p.pr_code = pr.pr_code  " +
                "        LEFT OUTER JOIN (SELECT * FROM sale WHERE sa_end_date >= SYSDATE AND sa_date <= SYSDATE) s ON p.pr_code = s.pr_code " +
                "        WHERE ca_code IN (  " +
                "            SELECT ca_code  " +
                "            FROM category  " +
                "            WHERE ca_topcode IN ( " +
                "                SELECT ca_code  " +
                "                FROM category  " +
                "                WHERE ca_topcode = ?  " +
                "            )  " +
                "        )  " +
                "    ORDER BY p.pr_view DESC  " +
                ")  " +
                "WHERE ROWNUM <= 20 ";
    
        PreparedStatement pstmt = null;
        ResultSet rs = null;
        try {
            pstmt = connection.prepareStatement(sql);
            pstmt.setString(1, ca_code);
            rs = pstmt.executeQuery();
            ProductBrandPriceDTO prDTO = null;
            if (rs.next()){
                list = new ArrayList<>();
                do {
                    prDTO = new ProductBrandPriceDTO();
                    prDTO.setPr_code(rs.getString("pr_code"));
                    prDTO.setPr_name(rs.getString("pr_name"));
                    prDTO.setPr_name(rs.getString("pr_name"));
                    prDTO.setPrpri_price(rs.getInt("price"));
                    prDTO.setSa_rate(rs.getDouble("sale_rate"));
                    prDTO.setRealPrice(rs.getInt("realprice"));
                    prDTO.setBr_name(rs.getString("br_name"));
                    prDTO.setPrm_url(rs.getString("prm_url"));
                    list.add(prDTO);
                } while (rs.next());
            }
        } catch (SQLException e) {
            throw new RuntimeException(e);
        } finally {
            try {
                rs.close();
                pstmt.close();
            } catch (SQLException e) {
                throw new RuntimeException(e);
            }
    
        }
        return list;
    }
    ```
    
- 카테고리의 자식 카테고리 불러오는 DAO 보기
    
    ```java
    @Override
    public List<CategoryDTO> selectMSCategory(Connection connection, String ca_code) {
        List<CategoryDTO> list = null;
        PreparedStatement pstmt1 = null;
        PreparedStatement pstmt2 = null;
        ResultSet rs1 = null;
        ResultSet rs2 = null;
    
        String sql1 = "SELECT * FROM category WHERE ca_code = ? ";
        String sql2 = "SELECT * FROM category WHERE ca_topcode = ? ";
    
        try {
            pstmt1 = connection.prepareStatement(sql1);
            pstmt1.setString(1,ca_code);
            rs1 = pstmt1.executeQuery();
            if (rs1.next()){
                list = new ArrayList<>();
                CategoryDTO dto = new CategoryDTO();
                dto.setCa_code(rs1.getString("ca_code"));
                dto.setCa_name(rs1.getString("ca_name"));
                dto.setCa_topcode(rs1.getString("ca_topcode"));
                dto.setCa_level(rs1.getInt("ca_level"));
                list.add(dto);
                pstmt2 = connection.prepareStatement(sql2);
                pstmt2.setString(1,ca_code);
                rs2 = pstmt2.executeQuery();
                if (rs2.next()){
                    do {
                        dto = new CategoryDTO();
                        dto.setCa_code(rs2.getString("ca_code"));
                        dto.setCa_name(rs2.getString("ca_name"));
                        dto.setCa_topcode(rs2.getString("ca_topcode"));
                        dto.setCa_level(rs2.getInt("ca_level"));
                        list.add(dto);
                    } while (rs2.next());
                }
            }
        } catch (SQLException e) {
            throw new RuntimeException(e);
        } finally {
            try {
                rs1.close();
                rs2.close();
                pstmt1.close();
                pstmt2.close();
            } catch (SQLException e) {
                throw new RuntimeException(e);
            }
        }
    
        return list;
    }
    ```
    

### 제품 검색

![스크린샷 2022-11-30 오후 11.25.02.png](/ect/Olive%20Zero%201d9513c348cf45d7a9f379a247ec399c/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2022-11-30_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE_11.25.02.png)

![스크린샷 2022-11-30 오후 11.25.18.png](/ect/Olive%20Zero%201d9513c348cf45d7a9f379a247ec399c/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2022-11-30_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE_11.25.18.png)

- 제품 이름에 포함되는 키워드 입력시 해당 제품 리스트 가져오는 기능
- 제품 검색 DAO 보기
    
    ```java
    public List<ProductBrandPriceDTO> selectSearchProduct(Connection connection, String keyword){
        String realKeyword = "*" + keyword.replaceAll("\\s", "");
        System.out.println(realKeyword);
        List<ProductBrandPriceDTO> list = null;
        String sql = "SELECT p.*, b.br_name, pi.prm_url , NVL(pr.prpri_price, 0) price, NVL(s.sa_rate, 0) sale_rate, NVL(pr.prpri_price, 0)*(1-NVL(s.sa_rate, 0)) realprice " +
                "FROM product p " +
                "    JOIN brand b ON p.br_code = b.br_code  " +
                "    LEFT OUTER JOIN productmimg pi ON p.pr_code = pi.pr_code  " +
                "    LEFT OUTER JOIN (SELECT * FROM prprice WHERE prpri_enddate IS NULL) pr ON p.pr_code = pr.pr_code " +
                "    LEFT OUTER JOIN (SELECT * FROM sale WHERE sa_end_date >= SYSDATE AND sa_date <= SYSDATE) s ON p.pr_code = s.pr_code " +
                "WHERE REGEXP_LIKE(replace(trim(p.pr_name),' ', ''), ?, 'i') ";
        PreparedStatement pstmt = null;
        ResultSet rs = null;
        try {
            pstmt = connection.prepareStatement(sql);
            pstmt.setString(1 , realKeyword);
            rs = pstmt.executeQuery();
            if (rs.next()){
                list = new ArrayList<>();
                ProductBrandPriceDTO dto = null;
                do {
                    dto = new ProductBrandPriceDTO();
                    dto.setPr_code(rs.getString("pr_code"));
                    dto.setPr_name(rs.getString("pr_name"));
                    dto.setPrpri_price(rs.getInt("price"));
                    dto.setSa_rate(rs.getDouble("sale_rate"));
                    dto.setRealPrice(rs.getInt("realprice"));
                    dto.setBr_name(rs.getString("br_name"));
                    dto.setPrm_url(rs.getString("prm_url"));
                    list.add(dto);
                } while (rs.next());
            }
        } catch (SQLException e) {
            throw new RuntimeException(e);
        } finally {
            JdbcUtil.close(rs);
            JdbcUtil.close(pstmt);
        }
        return list;
    }
    ```
