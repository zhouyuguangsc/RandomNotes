由于.Net8 增加了密钥长度的检查，所以考虑跳过检查，方案如下：
```
services.AddJwt<JwtHandler>(enableGlobalAuthorize: true, jwtBearerConfigure: options =>
{
    var a = new ShortHS256KeyCryptoProvider();
    CryptoProviderFactory.Default.CustomCryptoProvider = a;
    options.TokenValidationParameters.IssuerSigningKeyResolver = (tokenString, securityToken, identifier, parameters) =>
    {
        string alg = null;
        if (securityToken is JwtSecurityToken jwtSecurityToken)
            alg = jwtSecurityToken.SignatureAlgorithm;
        if (securityToken is Microsoft.IdentityModel.JsonWebTokens.JsonWebToken jsonWebToken)
            alg = jsonWebToken.Alg;
        if (parameters.IssuerSigningKey is SymmetricSecurityKey symIssKey
            && alg != null)
        {
            // workaround for breaking change in "System.IdentityModel.Tokens.Jwt 6.30.1+ 
            switch (alg?.ToLowerInvariant())
            {
                case "hs256":
                    return new[] { ExtendKeyLengthIfNeeded(symIssKey, 32) };
                case "hs512":
                    return new[] { ExtendKeyLengthIfNeeded(symIssKey, 64) };
            }
        }
        return new[] { parameters.IssuerSigningKey };
    };

    SymmetricSecurityKey ExtendKeyLengthIfNeeded(SymmetricSecurityKey key, int minLenInBytes)
    {
        if (key != null && key.KeySize < (minLenInBytes * 8))
        {
            var newKey = new byte[minLenInBytes]; // zeros by default
            key.Key.CopyTo(newKey, 0);
            return new SymmetricSecurityKey(newKey);
        }
        return key;
    }
});

public class ShortHS256KeyCryptoProvider : ICryptoProvider
{
    public object Create(string algorithm, params object[] args)
    {
        if (args is null || args.Length == 0)
            return null;
        if (args[0] is SecurityKey)
            return null;
        if (!algorithm.Equals(SecurityAlgorithms.HmacSha256Signature, StringComparison.OrdinalIgnoreCase)
            && !algorithm.Equals(SecurityAlgorithms.HmacSha256, StringComparison.OrdinalIgnoreCase))
            return null;

        return new HMACSHA256(args[0] as byte[]);
    }

    public bool IsSupportedAlgorithm(string algorithm, params object[] args)
    {
        if(args is null || args.Length == 0)
            return false;
        if (args[0] is SecurityKey)
            return false;
        if(!algorithm.Equals(SecurityAlgorithms.HmacSha256Signature, StringComparison.OrdinalIgnoreCase)
            && !algorithm.Equals(SecurityAlgorithms.HmacSha256, StringComparison.OrdinalIgnoreCase))
            return false;
        return true;
    }

    public void Release(object cryptoInstance)
    {
        if (cryptoInstance is IDisposable disposableObject)
            disposableObject.Dispose();
    }
}
```

参考文档：

[错误码：IDX10720](https://github.com/AzureAD/azure-activedirectory-identitymodel-extensions-for-dotnet/wiki/IDX10720)

[StackOverflow：Migrating existing API project to .NET 8 getting error when creating JWT tokens](https://stackoverflow.com/questions/77518092/migrating-existing-api-project-to-net-8-getting-error-when-creating-jwt-tokens)

[Net8 Issues：uses the new JwtSecurityTokenHandler() WriteToken error](https://github.com/dotnet/aspnetcore/issues/52369)

[GitHub：Enforce key sizes when creating HMAC #2072](https://github.com/AzureAD/azure-activedirectory-identitymodel-extensions-for-dotnet/pull/2072)

[GitHub：Microsoft.IdentityModel.Tokens](https://github.com/AzureAD/azure-activedirectory-identitymodel-extensions-for-dotnet/tree/7.1.2)

[使用自定义 CryptoProvider](https://github.com/AzureAD/azure-activedirectory-identitymodel-extensions-for-dotnet/wiki/Using-a-custom-CryptoProvider)


