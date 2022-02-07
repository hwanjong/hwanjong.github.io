---
title: "Oauth 2.0 social sign-in refactoring"
date: 2022-02-08 00:26:28 -0400
categories: backend toy-project tart
tags : java oauth kakao google
toc : true
---

#개요
소셜로그인을 위하여 Oauth 2.0으로 백엔드 서버에 리다이렉트 할 경우 플랫폼의 종류와 상관없이 동작하도록 하며 확장이 용이하도록 리팩토링 한다.
`toy project`, `tart`, `social login`



# 1. Rest Controller - Factory Method
1) 소셜 플랫폼에서 리다이렉트 시 동일한 Callback 함수 실행.
2) Parameter로 전달 받은 Social Type을 이용하여 Factory Method로 객체 생성. *(플랫폼별 속성과 동작이 다르기 때문)*

``` java
   @RequestMapping(value = "oauth", method = RequestMethod.GET)
   public HashMap<String, String> registerUserFromSNS(HttpServletResponse response, String code, String error, String social) {

       HashMap<String,String> responseBody = new HashMap<>();
       //create social service with social type.
       SocialService externService = SocialService.create(social);
       if (error != null) {
           response.setStatus(HttpServletResponse.SC_FORBIDDEN);
           responseBody.put(TartBeanProperty.TART_ENUM.RESPONSE_TEXT.name(),error);
           return responseBody;
       }

       String token = externService.getAccessToken(code, TartBeanProperty.REDIRECT_URI_LOGIN);
       UserInfo userInfo = externService.getUserInfo(token);
       userDao.updateFromSocial(userInfo);

       return responseBody;
   }
```

``` java
public abstract class SocialService {

    public String CLIENT_ID;
    public String CLIENT_SECRET;
    public String BASE_TOKEN_URL;
    public String GET_USERINFO_URL;
    public String UNLINK_URL;

    public TartBeanProperty.SOCIAL social;

    public static SocialService create(String social){
        if(TartBeanProperty.SOCIAL.KAKAO.name().equals(social)){
            return new KakaoService();
        }else if(TartBeanProperty.SOCIAL.GOOGLE.name().equals(social)){
            return new GoogleService();
        }
        return null;
    }
```
``` java
    public enum SOCIAL{
        KAKAO, GOOGLE
    }
```

# 2. Template Method
- Super Class에서 공통 인터페이스를 구현하고 Template Method로 플랫폼별 다른 동작을 Sub class에 위임함. (*Header와 Property 차이와 플랫폼별로 가져오는 정보 차이*)
- parseUserInfo 는 각 subclass에서 구현

**abstract SocialService.class**
``` java
   public UserInfo getUserInfo(String token){
        String responseBody = "";
        try {
            RestTemplate restTemplate = new RestTemplate();

            HttpHeaders header = new HttpHeaders();
            header.add("Authorization", "Bearer " + token);
            header.add("Content-type", "application/x-www-form-urlencoded;charset=utf8");

            HttpEntity request = new HttpEntity(header);
            HttpEntity<String> response = restTemplate.exchange(
                    GET_USERINFO_URL,
                    HttpMethod.POST,
                    request,
                    String.class
            );
            responseBody = response.getBody();
            System.out.println("response body : " + responseBody);
        } catch (Exception e) {
            e.printStackTrace();
        }

        // Template Method, Delegate to subclass
        return parseUserInfo(responseBody);

    }

```
**SubClass 에서 플랫폼별 차이 구현**
``` java
public GoogleService() {
    super.social = TartBeanProperty.SOCIAL.GOOGLE;
    super.CLIENT_ID = "SOME_TEXT";
    super.CLIENT_SECRET = "SOME_TEXT";
    super.BASE_TOKEN_URL = "https://oauth2.googleapis.com/token";
    super.GET_USERINFO_URL = "https://www.googleapis.com/oauth2/v3/userinfo";
    super.UNLINK_URL = "https://oauth2.googleapis.com/revoke";
}

@Override
protected UserInfo parseUserInfo(String responseBody) {
    JsonElement element = new JsonParser().parse(responseBody);

    try {
        String id = element.getAsJsonObject().get("sub").getAsString();
        String email = element.getAsJsonObject().get("email").getAsString();
        String name = element.getAsJsonObject().get("name").getAsString();
        System.out.println("email: " + email + ", name : " + name);
    } catch (Exception e) {
        e.printStackTrace();
    }
    return null;
}

@Override
protected void updateUrlForToken(String token) {
    UNLINK_URL += "?token=" + token;
}
```



# 3. Test Code
- Junit5, mockito.
``` java
//junit5 test code...
```
