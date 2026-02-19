# Chapter 14 — Advanced Shaders

---

## 1. Procedural Shaders (No Textures Required)

### 1.1 Checkerboard Pattern
```glsl
vec3 checkerboard(vec2 uv, float scale) {
    vec2 grid = floor(uv * scale);
    float checker = mod(grid.x + grid.y, 2.0);
    return mix(vec3(0.8), vec3(0.2), checker);
}
// MANUAL TRACE: uv = (0.3, 0.7), scale = 4
//   grid = floor(1.2, 2.8) = (1, 2)
//   checker = mod(1 + 2, 2) = mod(3, 2) = 1
//   color = mix(white, black, 1) = black
```

### 1.2 Circle / SDF (Signed Distance Field)
```glsl
float circleSDF(vec2 uv, vec2 center, float radius) {
    return length(uv - center) - radius;
    // Negative = inside, Zero = on edge, Positive = outside
}

void main() {
    vec2 uv = gl_FragCoord.xy / u_resolution;
    float d = circleSDF(uv, vec2(0.5), 0.3);
    
    vec3 color = vec3(smoothstep(0.01, -0.01, d)); // Anti-aliased edge!
    fragColor = vec4(color, 1.0);
}
```

---

## 2. Post-Processing Shaders

### 2.1 Full-Screen Quad Setup
```
POST-PROCESSING PIPELINE:
═════════════════════════
  STEP 1: Render scene to a Framebuffer texture (not the screen).
  STEP 2: Bind that texture.
  STEP 3: Draw a full-screen triangle covering the entire viewport.
  STEP 4: Fragment shader reads the scene texture and applies effects.
```

### 2.2 Grayscale
```glsl
vec4 sceneColor = texture(u_sceneTexture, v_texCoord);
float luminance = dot(sceneColor.rgb, vec3(0.2126, 0.7152, 0.0722));
fragColor = vec4(vec3(luminance), 1.0);
```

### 2.3 Gaussian Blur (Separable Two-Pass)
```glsl
// Horizontal pass
vec3 blur(sampler2D tex, vec2 uv, vec2 texelSize) {
    vec3 result = vec3(0.0);
    float weights[5] = float[](0.227, 0.194, 0.121, 0.054, 0.016);
    
    result += texture(tex, uv).rgb * weights[0];
    for (int i = 1; i < 5; i++) {
        result += texture(tex, uv + vec2(texelSize.x * float(i), 0.0)).rgb * weights[i];
        result += texture(tex, uv - vec2(texelSize.x * float(i), 0.0)).rgb * weights[i];
    }
    return result;
}
// Run again with Y direction for vertical pass.
// 2-pass separable: 10 samples instead of 25 (5×5 kernel).
```

### 2.4 Vignette
```glsl
float vignette(vec2 uv) {
    float dist = distance(uv, vec2(0.5));
    return 1.0 - smoothstep(0.4, 0.8, dist);
}
```

---

## 3. Shadow Mapping (Basic Implementation)

### 3.1 Shadow Fragment Shader
```glsl
// In the camera's fragment shader
uniform sampler2D u_shadowMap;
uniform mat4 u_lightSpaceMatrix;

float calculateShadow(vec4 fragPosLightSpace) {
    // Perspective divide + NDC → [0,1]
    vec3 projCoords = fragPosLightSpace.xyz / fragPosLightSpace.w;
    projCoords = projCoords * 0.5 + 0.5;
    
    float closestDepth = texture(u_shadowMap, projCoords.xy).r;
    float currentDepth = projCoords.z;
    
    // Bias to prevent shadow acne
    float bias = max(0.05 * (1.0 - dot(N, L)), 0.005);
    
    float shadow = currentDepth - bias > closestDepth ? 1.0 : 0.0;
    return shadow;
}

void main() {
    vec4 fragLightSpace = u_lightSpaceMatrix * vec4(v_worldPos, 1.0);
    float shadow = calculateShadow(fragLightSpace);
    
    vec3 lighting = ambient + (1.0 - shadow) * (diffuse + specular);
    fragColor = vec4(lighting * u_objectColor, 1.0);
}
```

---

## 4. PBR Basics in Fragment Shader

*See Advanced Rendering Modules 1-2 for the complete PBR derivation.*

```glsl
// Minimal PBR: Cook-Torrance specular + Lambertian diffuse
void main() {
    vec3 N = normalize(v_normal);
    vec3 V = normalize(u_cameraPos - v_worldPos);
    vec3 L = normalize(u_lightPos - v_worldPos);
    vec3 H = normalize(V + L);
    
    vec3 F0 = mix(vec3(0.04), albedo, metallic);
    
    float D = DistributionGGX(N, H, roughness);
    float G = GeometrySmith(N, V, L, roughness);
    vec3  F = fresnelSchlick(max(dot(H, V), 0.0), F0);
    
    vec3 kD = (vec3(1.0) - F) * (1.0 - metallic);
    vec3 specular = (D * G * F) / max(4.0 * max(dot(N,V), 0.0) * max(dot(N,L), 0.0), 0.001);
    
    float NdotL = max(dot(N, L), 0.0);
    vec3 Lo = (kD * albedo / PI + specular) * lightColor * NdotL;
    
    fragColor = vec4(Lo + vec3(0.03) * albedo, 1.0);
}
```
