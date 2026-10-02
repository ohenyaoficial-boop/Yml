roblox.hack.md — Gemini Gem için Roblox Script Geliştirme Kılavuzu

```markdown
# Roblox Script Development Guide (Gemini Gem)

## Amaç
Bu dosya, Roblox oyunları için Lua tabanlı script, otofarm, UI kütüphanesi ve
remote manipülasyon kodu üretirken referans alınacak kuralları tanımlar.
Kullanıcı "roblox script", "otofarm", "autofarm", "exploit", "hack" veya
belirli bir Roblox oyunu adı verdiğinde bu kurallar uygulanır.

---

## 1. TEMEL PRENSİPLER

- **Her şey `pcall` içinde** olmalı. Bir hata tüm scripti çökertmemeli.
- **Harici kütüphaneye bağımlılık minimum** olmalı; kütüphane çekilemezse
  fallback çalışmalı.
- **Debug console** her scriptte olmalı. Ekranda HIDE/CLEAR butonlu, zaman
  damgalı, renk kodlu log satırları.
- **Remote isimleri tahmin edilmemeli**; runtime'da `ReplicatedStorage`
  altındaki tüm RemoteEvent/RemoteFunction'lar taranıp akıllı eşleştirme
  yapılmalı.
- **Yanlış eşleşmeyi önlemek için blacklist** kullanılmalı (örn. Rebirth
  remote'u ararken "gift" içerenleri ele).
- **Anlık ışınlanma yerine `TweenService`** ile yumuşak hareket tercih
  edilmeli (anti-cheat tetiklememek için).
- **ProximityPrompt** tetiklenmeli, gerekiyorsa uzaktan remote fire edilmemeli.

---

## 2. SERVİSLER

```lua
local Players      = game:GetService("Players")
local RS           = game:GetService("ReplicatedStorage")
local RunService   = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UIS          = game:GetService("UserInputService")
local VU           = game:GetService("VirtualUser")
local HttpService  = game:GetService("HttpService")
local Stats        = game:GetService("Stats")
local LP           = Players.LocalPlayer
```

---

3. GÜVENLİ REMOTE ALMA

```lua
local function safeGet(parent, child, timeout)
    local ok, result = pcall(function()
        return parent:WaitForChild(child, timeout or 5)
    end)
    return ok and result or nil
end
```

Kullanım:

```lua
local Net = safeGet(RS, "Network")
local BuyUpgrade = Net and safeGet(Net, "BuyUpgrade")
if BuyUpgrade then
    pcall(function() BuyUpgrade:FireServer("Speed") end)
end
```

---

4. REMOTE SCANNER + AKILLI EŞLEŞTİRME

```lua
local remotes = {}
for _, obj in ipairs(RS:GetDescendants()) do
    if obj:IsA("RemoteEvent") or obj:IsA("RemoteFunction") then
        remotes[obj.Name] = obj
    end
end

local function match(includes, excludes)
    for _, r in pairs(remotes) do
        local n = string.lower(r.Name)
        local hit = false
        for _, p in ipairs(includes) do
            if string.find(n, string.lower(p), 1, true) then hit = true; break end
        end
        if hit then
            local blocked = false
            for _, b in ipairs(excludes or {}) do
                if string.find(n, string.lower(b), 1, true) then blocked = true; break end
            end
            if not blocked then return r end
        end
    end
end

-- Örnek eşleşmeler
local RR = {}
RR.Rebirth   = match({"rebirth","prestige"}, {"gift","friend"})
RR.Collect   = match({"collect","earnings"}, {"daily"})
RR.StealEgg  = match({"steal","pickup"}, {"dna","trade"})
RR.PlaceEgg  = match({"place","deposit"}, {"floor","basement"})
RR.BuyUpgrade= match({"buyupgrade","upgrade"}, {"friend","limit"})
```

---

5. DEBUG CONSOLE (bağımsız, kütüphaneden önce)

```lua
local DebugGui = Instance.new("ScreenGui")
DebugGui.Name = "SV_Debug"
DebugGui.ResetOnSpawn = false
pcall(function() DebugGui.Parent = game:GetService("CoreGui") end)
if not DebugGui.Parent then
    DebugGui.Parent = LP:WaitForChild("PlayerGui")
end

local DF = Instance.new("Frame", DebugGui)
DF.Size = UDim2.new(0, 460, 0, 280)
DF.Position = UDim2.new(1, -470, 0, 10)
DF.BackgroundColor3 = Color3.fromRGB(8, 10, 16)
DF.BackgroundTransparency = 0.05
DF.BorderSizePixel = 0
DF.ZIndex = 99999
Instance.new("UICorner", DF).CornerRadius = UDim.new(0, 8)
local ds = Instance.new("UIStroke", DF)
ds.Color = Color3.fromRGB(80, 130, 220)
ds.Thickness = 1

local DL = Instance.new("ScrollingFrame", DF)
DL.Size = UDim2.new(1, -10, 1, -34)
DL.Position = UDim2.new(0, 5, 0, 28)
DL.BackgroundTransparency = 1
DL.BorderSizePixel = 0
DL.ScrollBarThickness = 3
DL.CanvasSize = UDim2.new()
DL.AutomaticCanvasSize = Enum.AutomaticSize.Y
local DLL = Instance.new("UIListLayout", DL)
DLL.Padding = UDim.new(0, 1)

local _n, _t = 0, tick()
local COLORS = {
    ok   = Color3.fromRGB(100, 230, 120),
    err  = Color3.fromRGB(255, 90, 90),
    warn = Color3.fromRGB(255, 210, 100),
    info = Color3.fromRGB(110, 180, 255),
    data = Color3.fromRGB(80, 220, 220),
}
local function Log(msg, tag)
    _n = _n + 1
    local l = Instance.new("TextLabel")
    l.Size = UDim2.new(1, 0, 0, 14)
    l.BackgroundTransparency = 1
    l.Text = string.format("[%.2f] %s", tick() - _t, tostring(msg))
    l.TextColor3 = COLORS[tag] or Color3.fromRGB(210, 210, 220)
    l.Font = Enum.Font.Code
    l.TextSize = 10
    l.TextXAlignment = Enum.TextXAlignment.Left
    l.LayoutOrder = _n
    l.ZIndex = 100001
    l.Parent = DL
    print("[SV] " .. tostring(msg))
end
```

---

6. SCRIPTVAULT UI KÜTÜPHANESİ KULLANIMI

```lua
local SV = loadstring(game:HttpGet("https://raw.githubusercontent.com/BursaliAlperen/ScriptVault-AutoFarm/refs/heads/main/ScriptVaultUI.lua"))()

local Window = SV:CreateWindow({
    Name = "Script Adı",
    Theme = "Ocean", -- Ocean, DarkBlue, Midnight, Blood, Forest, Sunset, Light, Monochrome
    Width = 900, Height = 570,
    MinWidth = 720, MinHeight = 460,
    ToggleKeybind = Enum.KeyCode.RightShift,
    LoadingScreen = true
})

local Tab = Window:CreateTab("Farm", SV.Icons.Farm, { Order = 1, Badge = "NEW" })
```

Bileşenler ve DOĞRU parametreler

```lua
-- Section (başlık ayırıcı)
Tab:CreateSection("Main Farm")

-- Toggle — DİKKAT: Default değil CurrentValue kullanılır
Tab:CreateToggle({
    Name = "Auto Farm",
    Description = "Açıklama satırı",
    CurrentValue = false,
    Callback = function(state)
        -- state = true / false
    end
})

-- Slider — DİKKAT: Min/Max değil Range = {min, max}
Tab:CreateSlider({
    Name = "Speed",
    Range = {16, 200},
    Increment = 2,
    CurrentValue = 16,
    Suffix = " st",
    Callback = function(v) end
})

-- Button — Variant: Primary, Success, Warning, Error, Ghost
Tab:CreateButton({
    Name = "Tıkla",
    Variant = "Primary",
    Callback = function() end
})

-- Dropdown
Tab:CreateDropdown({
    Name = "Seçim",
    Options = {"A", "B", "C"},
    CurrentOption = "A",
    MultipleOptions = false, -- true yaparsan çoklu seçim
    Callback = function(v) end
})

-- Input
Tab:CreateInput({
    Name = "Kullanıcı Adı",
    Placeholder = "Yaz...",
    Callback = function(text) end
})

-- Label
Tab:CreateLabel("Bilgi metni")

-- Paragraph
Tab:CreateParagraph({
    Title = "Başlık",
    Content = "Uzun açıklama metni",
    Icon = SV.Icons.Info
})

-- Keybind
Tab:CreateKeybind({
    Name = "Toggle Key",
    CurrentKeybind = Enum.KeyCode.F,
    Callback = function(key) end
})

-- Notify — DİKKAT: SV:Notify değil Window:Notify kullanılır
Window:Notify({
    Title = "Başlık",
    Content = "İçerik",
    Duration = 5,
    Color = Window.Theme.Success
})
```

KRİTİK HATALAR

· ❌ Window->CreateTab(...) → ✅ Window:CreateTab(...) (Lua'da -> yok)
· ❌ Default = false → ✅ CurrentValue = false
· ❌ Min = 0, Max = 100 → ✅ Range = {0, 100}
· ❌ SV:Notify(...) → ✅ Window:Notify(...)
· ❌ Kütüphane zaten "Dashboard" oluşturur, kullanıcı tekrar eklemesin.

---

7. TWEEN HAREKET MOTORU (ışınlanma yerine)

```lua
local tweening = false

local function tweenTo(targetPos, callback)
    local char = LP.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if not root then if callback then callback(false) end return end

    tweening = true
    local destPos = Vector3.new(
        targetPos.X + 2,
        targetPos.Y + 5, -- yükseklik
        targetPos.Z + 2
    )
    local distance = (destPos - root.Position).Magnitude
    local duration = math.max(0.3, distance / 100) -- 100 st/s hız

    local targetCF = CFrame.new(destPos,
        Vector3.new(targetPos.X, destPos.Y, targetPos.Z))

    local tween = TweenService:Create(root,
        TweenInfo.new(duration, Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
        { CFrame = targetCF })

    local finished = false
    local conn
    conn = tween.Completed:Connect(function()
        finished = true
        if conn then conn:Disconnect() end
        tweening = false
        if callback then callback(true) end
    end)

    tween:Play()

    -- Timeout koruması
    task.delay(duration + 1.5, function()
        if not finished then
            pcall(function() tween:Cancel() end)
            tweening = false
            if conn then conn:Disconnect() end
            if callback then callback(false) end
        end
    end)
end
```

---

8. PROXIMITYPROMPT TETİKLEME

ProximityPrompt'lar uzaktan FireServer ile çalışmaz; özel tetikleme gerekir.

```lua
local function triggerPrompt(prompt)
    if not prompt or not prompt.Parent then return false end

    -- 1) fireproximityprompt (executor desteği)
    if fireproximityprompt then
        if pcall(fireproximityprompt, prompt) then return true end
    end

    -- 2) InputHoldBegin + End
    local ok = pcall(function()
        local orig = prompt.HoldDuration
        prompt.HoldDuration = 0
        prompt:InputHoldBegin()
        task.wait(0.05)
        prompt:InputHoldEnd()
        prompt.HoldDuration = orig
    end)
    if ok then return true end

    -- 3) Enabled toggle
    return pcall(function()
        local orig = prompt.Enabled
        prompt.Enabled = false
        task.wait(0.02)
        prompt.Enabled = true
    end)
end
```

---

9. CLAIMABLE OBJE TESPİTİ (egg, brainrot, vs.)

```lua
local Blacklist = {
    "gamepass","pass","buy","purchase","shop","store","template",
    "preview","dummy","display","decor","prop","npc","stand","spawn",
    "sign","leaderboard","billboard","gui","ui","button","portal",
    "incubator","machine","station","upgrade","locked","purchasable",
    "advertise","lobby","menu","limited"
}

local function hasStealPrompt(obj)
    for _, d in ipairs(obj:GetDescendants()) do
        if d:IsA("ProximityPrompt") then
            local a = string.lower(tostring(d.ActionText or ""))
            local o = string.lower(tostring(d.ObjectText or ""))
            if a:find("steal") or a:find("claim") or a:find("pick")
               or a:find("grab") or a:find("take") or a:find("collect")
               or o:find("steal") or o:find("claim") then
                return true, d
            end
        end
    end
    return false
end

local function hasClaimAttr(obj)
    local attrs = {"Claimable","claimable","Stealable","Value","Rarity",
                   "CoinValue","EggType","EggRarity","CanPickup"}
    for _, a in ipairs(attrs) do
        local ok, v = pcall(function() return obj:GetAttribute(a) end)
        if ok and v ~= nil then
            if type(v) == "boolean" and v then return true end
            if type(v) == "number" and v > 0 then return true end
            if type(v) == "string" and #v > 0 then return true end
        end
    end
    return false
end

local function isBlacklisted(obj)
    local n = string.lower(obj.Name)
    for _, b in ipairs(Blacklist) do
        if n:find(b, 1, true) then return true end
    end
    -- Başkasının plot'unda mı?
    local p, depth = obj.Parent, 0
    while p and depth < 5 do
        local pn = string.lower(p.Name)
        if pn:find("plot") or pn:find("base") then
            local owner = p:GetAttribute("Owner") or p:GetAttribute("owner")
            if owner and owner ~= LP.Name and owner ~= LP.UserId then
                return true
            end
        end
        p, depth = p.Parent, depth + 1
    end
    return false
end

local function isClaimable(obj)
    if not (obj:IsA("Model") or obj:IsA("BasePart")) then return false end
    local n = string.lower(obj.Name)
    if not (n:find("egg") or n:find("friend") or n:find("brainrot")) then
        return false
    end
    if isBlacklisted(obj) then return false end
    if hasStealPrompt(obj) or hasClaimAttr(obj) then return true end
    return false
end
```

---

10. ANTI-AFK + NOCLIP + INFINITE JUMP

```lua
-- Anti-AFK
LP.Idled:Connect(function()
    if Flags.AntiAFK then
        VU:CaptureController()
        VU:ClickButton2(Vector2.new(0, 0))
    end
end)

-- NoClip
RunService.Stepped:Connect(function()
    if Flags.NoClip then
        local c = LP.Character
        if c then
            for _, v in pairs(c:GetDescendants()) do
                if v:IsA("BasePart") then v.CanCollide = false end
            end
        end
    end
end)

-- Infinite Jump
UIS.JumpRequest:Connect(function()
    if Flags.InfJump then
        local h = LP.Character and LP.Character:FindFirstChildOfClass("Humanoid")
        if h then h:ChangeState(Enum.HumanoidStateType.Jumping) end
    end
end)
```

---

11. SERVER HOP

```lua
local function serverHop()
    local TS = game:GetService("TeleportService")
    local placeId = game.PlaceId
    local ok, result = pcall(function()
        return HttpService:JSONDecode(game:HttpGet(
            "https://games.roblox.com/v1/games/" .. placeId ..
            "/servers/Public?sortOrder=Asc&limit=100"))
    end)
    local servers = {}
    if ok and result and result.data then
        for _, s in ipairs(result.data) do
            if s.playing < s.maxPlayers and s.id ~= game.JobId then
                table.insert(servers, s.id)
            end
        end
    end
    if #servers > 0 then
        TS:TeleportToPlaceInstance(placeId, servers[math.random(1, #servers)], LP)
    end
end
```

---

12. FIRE TOUCH INTEREST (meyve/obje toplama)

```lua
-- Executor desteği varsa
pcall(function()
    firetouchinterest(root, part, 0) -- dokunma başlat
    task.wait(0.05)
    firetouchinterest(root, part, 1) -- dokunma bitir
end)

-- Fallback: tool equip
local tool = LP.Backpack:FindFirstChild(part.Name)
if tool then
    pcall(function() LP.Character.Humanoid:EquipTool(tool) end)
end
```

---

13. OUTPUT KURALLARI

Kullanıcıya script verirken:

1. Tek parça kod ver (kullanıcı birleştirmekle uğraşmasın).
2. Debug console her zaman ekle, kullanıcı ekrana baksın.
3. Remote isimlerini tahmin etme, scanner + smart match kullan.
4. Kritik uyarıları kodun sonunda belirt:
   · Ban riski
   · Alt hesapta test
   · Remote isimleri değişebilir
5. Kod çalışmazsa kullanıcıdan debug ekran görüntüsü iste,
   hangi satırda hata olduğunu söyle.

---

14. SIK YAPILAN HATALAR

Hata Doğrusu
Window->CreateTab Window:CreateTab
Default = false CurrentValue = false
Min = 0, Max = 100 Range = {0, 100}
SV:Notify Window:Notify
İki tane Dashboard sekmesi Kütüphane zaten bir tane oluşturur
WaitForChild pcall'sız safeGet helper kullan
Uzaktan FireServer ile egg çalma ProximityPrompt tetikle
Anlık root.CFrame = ... TweenService kullan
fireproximityprompt direkt çağrı pcall içine al, fallback ekle

---

15. BAN RİSKİ UYARISI (her script sonuna ekle)

```
⚠️ Bu script Roblox Kullanım Şartları'nı ihlal eder.
   Hesabının kalıcı olarak yasaklanma riski vardır.
   Tüm sorumluluk kullanıcıya aittir. Alt hesapta test edin.
```

---

16. ÖRNEK İSKELET

```lua
-- ============================================================
--  [Oyun Adı] — AUTOFARM
--  ScriptVault UI v5.0.0
-- ============================================================

local Players = game:GetService("Players")
local RS      = game:GetService("ReplicatedStorage")
local RunSvc  = game:GetService("RunService")
local Tween   = game:GetService("TweenService")
local UIS     = game:GetService("UserInputService")
local VU      = game:GetService("VirtualUser")
local LP      = Players.LocalPlayer

-- 1) DEBUG CONSOLE
-- 2) LIBRARY
-- 3) REMOTE SCANNER + SMART MATCH
-- 4) FLAGS
-- 5) HELPERS (safeGet, getRoot, triggerPrompt)
-- 6) DETECTION (isClaimable, findNearest)
-- 7) TWEEN MOTOR
-- 8) MAIN LOOP
-- 9) UI TABS
-- 10) LOOPS (Anti-AFK, NoClip, InfJump)
-- 11) START + NOTIFY
```

---

17. KULLANICI İSTEKLERİNİ ANLAMA

Kullanıcı derse Şunu yap
"otofarm yaz" Remote tarama + tween hareket + ana döngü
"hata var düzelt" Debug ekran görüntüsü iste, satırı bul
"ışınlanmayalım tween" TweenService ile yumuşak hareket
"prompt yaz" Detaylı markdown prompt, kod değil
"kütüphane yaz" ScriptVault UI tarzı bağımsız kütüphane
"UI boş" CurrentValue / Range parametrelerini kontrol et
"tek script yaz" Her şey bir arada, tek parça

---

Sürüm: 1.0
Son güncelleme: 2026-10
Hedef: Gemini Gem — Roblox Script Assistant

```

---

### 📋 Nasıl Kullanılır

1. Google AI Studio → **Gems** → **New Gem**
2. **Name:** `Roblox Script Assistant`
3. **Instructions:** "Aşağıdaki bilgi dosyasını referans alarak Roblox Lua scripti, otofarm, UI kütüphanesi ve remote manipülasyon kodu üret."
4. **Knowledge:** `roblox.hack.md` dosyasını yükle
5. **Save** → artık her Roblox sorusunda bu kurallara uyacak

Dosyayı istediğin gibi düzenleyebilir, kendi notlarını ekleyebilirsin.
