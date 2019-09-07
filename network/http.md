### Message Body & Length
https://tools.ietf.org/html/rfc2616#section-4.3

4.3 Message Body

   The message-body (if any) of an HTTP message is used to carry the
   entity-body associated with the request or response. The message-body
   differs from the entity-body only when a transfer-coding has been
   applied, as indicated by the Transfer-Encoding header field (section
   14.41).

       message-body = entity-body
                    | <entity-body encoded as per Transfer-Encoding>

   Transfer-Encoding MUST be used to indicate any transfer-codings
   applied by an application to ensure safe and proper transfer of the
   message. Transfer-Encoding is a property of the message, not of the entity,
   and thus MAY be added or removed by any application along the
   request/response chain. (However, section 3.6 places restrictions on
   when certain transfer-codings may be used.)

   The rules for when a message-body is allowed in a message differ for
   requests and responses.

   The presence of a message-body in a request is signaled by the
   inclusion of a Content-Length or Transfer-Encoding header field in
   the request's message-headers. A message-body MUST NOT be included in
   a request if the specification of the request method (section 5.1.1)
   does not allow sending an entity-body in requests. A server SHOULD
   read and forward a message-body on any request; if the request method
   does not include defined semantics for an entity-body, then the
   message-body SHOULD be ignored when handling the request.

   For response messages, whether or not a message-body is included with
   a message is dependent on both the request method and the response
   status code (section 6.1.1). All responses to the HEAD request method
   MUST NOT include a message-body, even though the presence of entity-
   header fields might lead one to believe they do. All 1xx
   (informational), 204 (no content), and 304 (not modified) responses
   MUST NOT include a message-body. All other responses do include a
   message-body, although it MAY be of zero length.

4.4 Message Length

   The transfer-length of a message is the length of the message-body as
   it appears in the message; that is, after any transfer-codings have
   been applied. When a message-body is included with a message, the
   transfer-length of that body is determined by one of the following
   (in order of precedence):

   1.Any response message which "MUST NOT" include a message-body (such
     as the 1xx, 204, and 304 responses and any response to a HEAD
     request) is always terminated by the first empty line after the
     header fields, regardless of the entity-header fields present in
     the message.

   2.If a Transfer-Encoding header field (section 14.41) is present and
     has any value other than "identity", then the transfer-length is
     defined by use of the "chunked" transfer-coding (section 3.6),
     unless the message is terminated by closing the connection.

   3.If a Content-Length header field (section 14.13) is present, its
     decimal value in OCTETs represents both the entity-length and the
     transfer-length. The Content-Length header field MUST NOT be sent
     if these two lengths are different (i.e., if a Transfer-Encoding header field is present).
     If a message is received with both a Transfer-Encoding header field and a Content-Length header field,
     the latter MUST be ignored.

   For compatibility with HTTP/1.0 applications, HTTP/1.1 requests
   containing a message-body MUST include a valid Content-Length header field unless the server is known to be HTTP/1.1 compliant.
   If a request contains a message-body and a Content-Length is not given,
   the server SHOULD respond with 400 (bad request) if it cannot
   determine the length of the message, or with 411 (length required) if
   it wishes to insist on receiving a valid Content-Length.

**위의 rfc내용을 요약하면 다음과 같다.**

- 요청 메시지
> 요청 메시지 본문이 존재한다는 것은 요청 메시지 헤더에 Content-Length 또는 Transfer-Encoding 헤더 필드를 포함시킴으로써 알 수 있다. 스펙으로 인해 요청 엔티티 본문을 보낼 수 없는 경우 메시지 본문을 요청에 포함시켜서는 안된다. 서버는 요청 메소드에 엔티티 본문에 대해 정의 된 의미가 포함되지 않은 경우 요청을 처리 할 때 메시지 본문을 무시해야 한다.

- 응답 메시지
> 응답 메시지의 경우 body가 메시지에 존재 하는 상황이 request method나 status code에 의존한다.

- 메시지 본문이 메시지에 포함 된 경우 해당 본문의 전송 길이는 다음 중 하나에 의해 결정된다.
  - HEAD 요청이거나 status code가 1xx, 204, 304응답인 경우에는 body가 포함되면 안된다. 항상 헤더필드 다음 빈줄로써 메시지가 끝난다.
  - Transfer-Encoding 헤더 필드가 존재하고 "identity"이외의 다른 값을 갖는 경우, 연결을 닫아서 메시지가 종료되지 않는 한 "chunked"전송 코딩을 사용하여 전송 길이가 정의된다.
  - Content-Length 헤더 필드가 존재하면 필드의 값은 entity length와 transfer length 모두를 의미한다. 만약 두 개의 length가 다르다면 Content-Length헤더가 없어야한다.(Transfer-Encoding 헤더가 존재하면 Content-Length헤더는 없어야함.) 만약 Transfer-Encoding와 Content-Length헤더가 동시에 메시지에 포함되어 있다면 Content-Length를 무시한다.

- HTTP/1.0 과 HTTP/1.1의 호환
  - 서버가 HTTP/1.1 통신 이라고 알지 않는 이상 message-body가 포함된 요청은 반드시 유효한 Content-Lengh 헤더 필드를 포함해야한다. 그렇지않으면 서버는 400(Bad reqeust)이나 411(length require)로 응답해야한다.
