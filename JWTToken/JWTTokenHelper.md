## JWT Token Helper

```java
public class JWTTokenHelper {
    private static final CLogger logger = LogFactory.getCLogger(JWTTokenHelper.class.getName());

    /**
     * Description: Getting the SignatureAlogorithm object instance.
     *
     * @return SignatureAlgorithm
     */
    public static SignatureAlgorithm getSignatureAlgorithm() {
        SignatureAlgorithm signatureAlgorithm = SignatureAlgorithm.valueOf(TOKEN_SIGNINGALGORITHM);

        return signatureAlgorithm;
    }

    /**
     * Description: This method generates the signingKey object based on the
     * signingAlgorithm and UTF-8 encodingCharSet.
     *
     * @param signatureAlgorithm
     * @param securityKey
     * @return Key
     * @throws UnsupportedEncodingException
     */
    public static Key getKey(SignatureAlgorithm signatureAlgorithm, String securityKey)
            throws UnsupportedEncodingException {
        // We will sign our JWT with our ApiKey secret
        byte[] apiKeySecretBytes = DatatypeConverter
                .parseBase64Binary(DatatypeConverter.printBase64Binary(securityKey.getBytes(TOKEN_ENCODINGCHARSET)));
        Key signingKey = new SecretKeySpec(apiKeySecretBytes, signatureAlgorithm.getJcaName());

        return signingKey;
    }

    /**
     * Description: GenerateToken method will generate the builder based on the JWT
     * apis which packs the custom claims and signing with algorithm and expiration.
     *
     * @param securityKey
     * @param ttlMillis
     * @param signatureAlgorithm
     * @param signingKey
     * @param payload
     * @return TDToken
     * @throws UnsupportedEncodingException
     */
    public static String generateTDToken(String securityKey, long ttlMillis, SignatureAlgorithm signatureAlgorithm,
                                         Key signingKey, String ssoId, String buidType, Object payload) throws UnsupportedEncodingException {
        // The JWT signature algorithm we will be using to sign the token

        long nowMillis = System.currentTimeMillis();
        Date now = new Date(nowMillis);
        long expMillis = nowMillis + ttlMillis;
        // System.out.println("GenerateToken Session ID : "
        // +request.getSession().getId());

        Date exp = new Date(expMillis);

        Claims claims = new DefaultClaims();
        claims.put("iss", TOKEN_INTERNALOPERATIONS); // issuer
        claims.put("jti", TOKEN_APP); // ID
        claims.put("sub", TOKEN_PARSING); // Sub
        claims.put("sessionID", ssoId);
        claims.put("buidType", buidType);

        if (!ObjectUtils.isEmpty(payload))
            claims.put("payload", payload);

        String builder = Jwts.builder().setIssuedAt(now).setClaims(claims).signWith(signatureAlgorithm, signingKey)
                .setExpiration(exp).compact();
        // logger.info("9 : " + (b - a)/1000);

        return builder;
    }

    /**
     * Description: Validating the claims into this method and if validation fails
     * this method will return the false as flag and if validation is successfull
     * this returns the true.
     *
     * @param claims
     * @return boolean
     */
    public static boolean validateTdToken(Claims claims) {
        boolean flag = false;

        if (null != claims) {
            if (!ObjectUtils.isEmpty(claims.getIssuer())
                    && !ObjectUtils.isEmpty(claims.getSubject())
                    && !ObjectUtils.isEmpty(claims.getId())) {

                if (claims.getIssuer().equals(TOKEN_INTERNALOPERATIONS) && claims.getId().equals(TOKEN_APP)
                        && claims.getSubject().equals(TOKEN_PARSING)) {
                    flag = true;
                }
            }
        }

        return flag;
    }

    public static void extractToken(Claims claims, TokenDataDTO tokenDataDTO) {
        if (!ObjectUtils.isEmpty(claims.get("sessionID")))
            tokenDataDTO.setSsoId((String) claims.get("sessionID"));

        if (!ObjectUtils.isEmpty(claims.get("buidType")))
            tokenDataDTO.setBuidType((String) claims.get("buidType"));

        if (!ObjectUtils.isEmpty(claims.get("payload")))
            tokenDataDTO.setPayload(objectToMap(claims.get("payload")));
    }

    /**
     * validateAndExtractToken
     * @param key
     * @param builder
     * @return
     */
    public static TokenDataDTO validateAndExtractToken(String key, String builder) {
        TokenDataDTO tokenDataDTO = new TokenDataDTO();

        try {
            if (!StringUtils.isEmpty(key) && !StringUtils.isEmpty(builder)) {
                Claims claims = parseToken(key, builder);

                boolean tokenValidity = validateTdToken(claims);
                tokenDataDTO.setValidToken(tokenValidity);

                if (tokenValidity)
                    extractToken(claims, tokenDataDTO);
            }
        } catch (Exception e) {
            logger.error("Cannot validateAndExtractToken, e: " + e);
        }

        return tokenDataDTO;
    }

    /**
     * Description: Method unpack the claims and set into a PruCustomClaims object.
     *
     * @param securityKey
     * @param builder
     * @return
     * @throws UnsupportedJwtException
     * @throws MalformedJwtException
     * @throws SignatureException
     * @throws IllegalArgumentException
     * @throws UnsupportedEncodingException
     */
    private static Claims parseToken(String securityKey, String builder) throws UnsupportedJwtException,
            MalformedJwtException, SignatureException, IllegalArgumentException, UnsupportedEncodingException {

        Claims claims = null;
        
        try {
            claims = (Claims) Jwts.parser()
                    .setSigningKey(DatatypeConverter
                            .parseBase64Binary(DatatypeConverter.printBase64Binary(securityKey.getBytes(TOKEN_ENCODINGCHARSET))))
                    .parseClaimsJws(builder).getBody();
        }
        // ADS-6896 remove the expiration validation temporarily to extract the info from expired token
        catch (ExpiredJwtException ex)
        {
            claims = ex.getClaims();
        }

        return claims;
    }
}
```