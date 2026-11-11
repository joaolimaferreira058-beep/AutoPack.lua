-- ================= CONFIG =================
local MAQUINA_NOME     = "PackMachine"
local USAR_PEDIR       = false
local PACKS            = {"Colossal", "Giant", "Big", "Medium", "Low"}
local TENTATIVAS_FALHA = 3
local MODOS = {
    {nome = "Normal", rodada = 0.3, entre = 0.10, retry = 0.15},
    {nome = "Rápido", rodada = 0.2, entre = 0.05, retry = 0.12},
    {nome = "Turbo",  rodada = 0.1, entre = 0.03, retry = 0.08},
}
local modoIdx = 2
-- ==========================================

local Players = game:GetService("Players")
local RS = game:GetService("ReplicatedStorage")
local UIS = game:GetService("UserInputService")
local TS = game:GetService("TweenService")
local TPS = game:GetService("TeleportService")
local VirtualUser = game:GetService("VirtualUser")
local player = Players.LocalPlayer

local packMachine = RS:WaitForChild("Remotes"):WaitForChild("PackMachine")
local comprar = packMachine:WaitForChild("Comprar")
local pedir = packMachine:FindFirstChild("Pedir")

-- cores
local VERM    = Color3.fromRGB(205, 30, 35)
local VERM_ESC= Color3.fromRGB(120, 15, 20)
local FUNDO   = Color3.fromRGB(24, 12, 14)
local FUNDO2  = Color3.fromRGB(40, 22, 24)
local VERDE   = Color3.fromRGB(60, 210, 90)
local CINZA   = Color3.fromRGB(150, 150, 160)
local CINZA_E = Color3.fromRGB(70, 60, 62)
local LARANJA = Color3.fromRGB(255, 150, 30)
local COR_PACK = {
    Low = Color3.fromRGB(40,170,60), Medium = Color3.fromRGB(150,60,220),
    Big = Color3.fromRGB(230,170,30), Giant = Color3.fromRGB(40,110,230),
    Colossal = Color3.fromRGB(220,40,40),
}
-- valores padrão (são atualizados com o que o jogo mostrar)
local PRECO = {Low="$100M", Medium="$250M", Big="$500M", Giant="$1B", Colossal="$10B"}
local SORTE = {Low=1, Medium=2, Big=4, Giant=8, Colossal=16}

-- ================= HELPERS =================
local function tween(o, t, props, style, dir)
    local tw = TS:Create(o, TweenInfo.new(t, style or Enum.EasingStyle.Quad, dir or Enum.EasingDirection.Out), props)
    tw:Play(); return tw
end
local function novo(classe, props, pai)
    local o = Instance.new(classe)
    for k, v in pairs(props) do o[k] = v end
    o.Parent = pai
    return o
end
local function canto(o, r)
    local c = Instance.new("UICorner"); c.CornerRadius = UDim.new(0, r); c.Parent = o
end
local function borda(o, cor, esp, tr)
    local s = Instance.new("UIStroke")
    s.Color = cor; s.Thickness = esp or 1; s.Transparency = tr or 0
    s.Parent = o; return s
end
local function gradiente(o, c1, c2)
    local g = Instance.new("UIGradient")
    g.Color = ColorSequence.new(c1, c2); g.Rotation = 90; g.Parent = o
end
local function texto(pai, txt, pos, tam, fonte, ts, cor, alinh)
    return novo("TextLabel", {
        Text = txt, Position = pos, Size = tam, BackgroundTransparency = 1,
        Font = fonte or Enum.Font.Gotham, TextSize = ts or 12,
        TextColor3 = cor or Color3.new(1,1,1),
        TextXAlignment = alinh or Enum.TextXAlignment.Left,
        TextTruncate = Enum.TextTruncate.AtEnd,
    }, pai)
end
local function chamar(remote, ...)
    if remote:IsA("RemoteFunction") then return remote:InvokeServer(...)
    else return remote:FireServer(...) end
end
local function arrastar(handle, alvo)
    local drag, ini, pos
    handle.InputBegan:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.Touch or i.UserInputType == Enum.UserInputType.MouseButton1 then
            drag = true; ini = i.Position; pos = alvo.Position
            i.Changed:Connect(function()
                if i.UserInputState == Enum.UserInputState.End then drag = false end
            end)
        end
    end)
    UIS.InputChanged:Connect(function(i)
        if drag and (i.UserInputType == Enum.UserInputType.Touch or i.UserInputType == Enum.UserInputType.MouseMovement) then
            local d = i.Position - ini
            alvo.Position = UDim2.new(pos.X.Scale, pos.X.Offset + d.X, pos.Y.Scale, pos.Y.Offset + d.Y)
        end
    end)
end

-- ================= ANTI AFK =================
player.Idled:Connect(function()
    VirtualUser:CaptureController()
    VirtualUser:ClickButton2(Vector2.new())
end)

-- ================= ESTADO / LÓGICA =================
local gui, lblStatus, lblCount
local ativo, finderAuto = false, false
local usarPedir = USAR_PEDIR
local compras = 0

local function status(txt, ok)
    if not lblStatus then return end
    lblStatus.Text = "status: " .. txt
    lblStatus.TextColor3 = ok == true and VERDE
        or ok == false and Color3.fromRGB(240,90,90) or CINZA
end

local function getMaquina()
    return workspace:FindFirstChild(MAQUINA_NOME, true)
end

local function tpMaquina(somenteSeLonge)
    local maq = getMaquina()
    if not maq then status("máquina não achada", false); return false end
    local char = player.Character or player.CharacterAdded:Wait()
    local root = char:WaitForChild("HumanoidRootPart")
    local alvo
    if maq:IsA("BasePart") then alvo = maq.Position
    else
        local p = maq:FindFirstChildWhichIsA("BasePart", true)
        if not p then status("sem parte na máquina", false); return false end
        alvo = p.Position
    end
    if somenteSeLonge and (root.Position - alvo).Magnitude < 15 then return true end
    root.CFrame = CFrame.new(alvo + Vector3.new(0, 4, 0))
    return true
end

local function comprarPack(pack)
    if usarPedir and pedir then
        local maq = getMaquina()
        if maq then pcall(chamar, pedir, maq) end
    end
    local ok, res = pcall(chamar, comprar, pack)
    if ok and res ~= false then
        compras += 1
        if lblCount then lblCount.Text = "compras ok: " .. compras end
        status(pack .. " -> " .. tostring(res), true)
        return true
    end
    status(pack .. " falhou: " .. tostring(res), false)
    return false
end

local function loopCompra()
    while ativo do
        local m = MODOS[modoIdx]
        tpMaquina(true)
        for _, pack in ipairs(PACKS) do
            if not ativo then break end
            local falhas = 0
            while ativo and falhas < TENTATIVAS_FALHA do
                if comprarPack(pack) then falhas = 0; task.wait(m.entre)
                else falhas += 1; task.wait(m.retry) end
            end
            task.wait(m.entre)
        end
        task.wait(m.rodada)
    end
end

-- lê estoque/preço/sorte direto da janela da Pack Machine
local function lerCartas()
    local dados, restock = {}, nil
    local pg = player:FindFirstChild("PlayerGui")
    if not pg then return dados, restock end
    for _, o in ipairs(pg:GetDescendants()) do
        if o:IsA("TextLabel") and not (gui and o:IsDescendantOf(gui)) then
            local t = o.Text
            local nome = t:match("^Pack (%a+)$")
            if nome then
                local c = o.Parent
                for _ = 1, 5 do
                    if not c or c == pg then break end
                    local estoques = {}
                    for _, d in ipairs(c:GetDescendants()) do
                        if d:IsA("TextLabel") and d.Text:find("em estoque") then
                            table.insert(estoques, d)
                        end
                    end
                    if #estoques == 1 then
                        local info = {stock = tonumber(estoques[1].Text:match("[Xx](%d+)")) or 0}
                        for _, d in ipairs(c:GetDescendants()) do
                            if d:IsA("TextLabel") then
                                if d.Text:match("^%$[%d%.,]+%a*$") then info.price = d.Text end
                                local s = d.Text:upper():match("(%d+)%s*X%s*SORTE")
                                if s then info.luck = s end
                            end
                        end
                        dados[nome] = info
                        break
                    end
                    c = c.Parent
                end
            end
            local r = t:match("Rep.-e em (.+)$")
            if r then restock = r end
        end
    end
    return dados, restock
end

-- ================= INTRO =================
local function intro()
    local g = novo("ScreenGui", {Name = "AutoPackIntro", ResetOnSpawn = false,
        IgnoreGuiInset = true, DisplayOrder = 100}, player:WaitForChild("PlayerGui"))
    local bg = novo("Frame", {Size = UDim2.new(1,0,1,0), BackgroundColor3 = Color3.fromRGB(10,4,6),
        BackgroundTransparency = 1, BorderSizePixel = 0}, g)
    local box = novo("Frame", {AnchorPoint = Vector2.new(0.5,0.5), Position = UDim2.new(0.5,0,0.5,0),
        Size = UDim2.new(0,280,0,120), BackgroundTransparency = 1}, bg)
    local sc = Instance.new("UIScale", box); sc.Scale = 0.5
    local logo = texto(box, "AUTO PACK HUB", UDim2.new(0,0,0,0), UDim2.new(1,0,0,50),
        Enum.Font.GothamBlack, 32, VERM, Enum.TextXAlignment.Center)
    logo.TextTransparency = 1
    local sub = texto(box, "PACK MACHINE", UDim2.new(0,0,0,50), UDim2.new(1,0,0,20),
        Enum.Font.GothamBold, 13, Color3.fromRGB(200,200,210), Enum.TextXAlignment.Center)
    sub.TextTransparency = 1
    local barBg = novo("Frame", {AnchorPoint = Vector2.new(0.5,0), Position = UDim2.new(0.5,0,0,85),
        Size = UDim2.new(0.8,0,0,6), BackgroundColor3 = Color3.fromRGB(40,25,28),
        BackgroundTransparency = 1, BorderSizePixel = 0}, box)
    canto(barBg, 6)
    local fill = novo("Frame", {Size = UDim2.new(0,0,1,0), BackgroundColor3 = VERM, BorderSizePixel = 0}, barBg)
    canto(fill, 6)
    local lbl = texto(box, "", UDim2.new(0,0,0,98), UDim2.new(1,0,0,18),
        Enum.Font.Gotham, 12, CINZA, Enum.TextXAlignment.Center)

    tween(bg, 0.4, {BackgroundTransparency = 0.15})
    tween(sc, 0.6, {Scale = 1}, Enum.EasingStyle.Back)
    tween(logo, 0.5, {TextTransparency = 0})
    task.wait(0.3)
    tween(sub, 0.5, {TextTransparency = 0})
    tween(barBg, 0.4, {BackgroundTransparency = 0})
    task.wait(0.3)
    for _, e in ipairs({{"iniciando anti-afk...", 0.35}, {"procurando a máquina...", 0.7}, {"carregando finder...", 1}}) do
        lbl.Text = e[1]
        tween(fill, 0.5, {Size = UDim2.new(e[2],0,1,0)})
        task.wait(0.55)
    end
    lbl.Text = "pronto!"
    task.wait(0.3)
    tween(sc, 0.5, {Scale = 1.4}, Enum.EasingStyle.Quad, Enum.EasingDirection.In)
    tween(logo, 0.5, {TextTransparency = 1}); tween(sub, 0.5, {TextTransparency = 1})
    tween(lbl, 0.5, {TextTransparency = 1}); tween(barBg, 0.5, {BackgroundTransparency = 1})
    tween(fill, 0.5, {BackgroundTransparency = 1}); tween(bg, 0.6, {BackgroundTransparency = 1})
    task.wait(0.65)
    g:Destroy()
end

intro()

-- ================= UI =================
gui = novo("ScreenGui", {Name = "AutoPackHub", ResetOnSpawn = false}, player:WaitForChild("PlayerGui"))

-- ---------- janela principal ----------
local main = novo("Frame", {Position = UDim2.new(0,12,0,8), Size = UDim2.new(0,310,0,230),
    BackgroundColor3 = FUNDO, BorderSizePixel = 0, ClipsDescendants = true}, gui)
canto(main, 10); borda(main, VERM, 2, 0)
local mainScale = Instance.new("UIScale", main); mainScale.Scale = 0

local header = novo("Frame", {Size = UDim2.new(1,0,0,34), BackgroundColor3 = VERM, BorderSizePixel = 0}, main)
gradiente(header, Color3.fromRGB(235,45,45), Color3.fromRGB(150,15,20))
texto(header, "Auto Pack Hub", UDim2.new(0,12,0,0), UDim2.new(1,-90,1,0), Enum.Font.GothamBlack, 18)

local function miniBtn(txt, xOff, cor)
    local b = novo("TextButton", {Size = UDim2.new(0,26,0,22), Position = UDim2.new(1,xOff,0,6),
        BackgroundColor3 = cor, Text = txt, TextColor3 = Color3.new(1,1,1),
        Font = Enum.Font.GothamBold, TextSize = 14}, header)
    canto(b, 6); return b
end
local btnMin = miniBtn("–", -62, Color3.fromRGB(90,20,25))
local btnX = miniBtn("X", -32, Color3.fromRGB(60,10,14))
arrastar(header, main)

local sidebar = novo("Frame", {Position = UDim2.new(0,6,0,40), Size = UDim2.new(0,84,0,184),
    BackgroundTransparency = 1}, main)
local content = novo("Frame", {Position = UDim2.new(0,96,0,38), Size = UDim2.new(0,208,0,188),
    BackgroundTransparency = 1}, main)

local pages, tabBtns = {}, {}
local function pagina(nome)
    local f = novo("Frame", {Size = UDim2.new(1,0,1,0), BackgroundTransparency = 1, Visible = false}, content)
    pages[nome] = f; return f
end
local function mostrar(nome)
    for n, f in pairs(pages) do f.Visible = (n == nome) end
    for n, b in pairs(tabBtns) do
        tween(b, 0.15, {BackgroundColor3 = (n == nome) and Color3.fromRGB(235,45,45) or VERM_ESC})
    end
end
local function aba(nome, y)
    local b = novo("TextButton", {Size = UDim2.new(1,0,0,38), Position = UDim2.new(0,0,0,y),
        BackgroundColor3 = VERM_ESC, Text = nome, TextColor3 = Color3.new(1,1,1),
        Font = Enum.Font.GothamBlack, TextSize = 14, AutoButtonColor = false}, sidebar)
    canto(b, 6); borda(b, Color3.fromRGB(255,80,80), 1.5, 0.4)
    b.MouseButton1Click:Connect(function() mostrar(nome) end)
    tabBtns[nome] = b
end
aba("Compra", 0); aba("Jogador", 46); aba("Finder", 92); aba("Outros", 138)

local function botao(pai, txt, y)
    local b = novo("TextButton", {Size = UDim2.new(1,-12,0,32), Position = UDim2.new(0,6,0,y),
        BackgroundColor3 = VERM, Text = txt, TextColor3 = Color3.new(1,1,1),
        Font = Enum.Font.GothamBlack, TextSize = 13}, pai)
    canto(b, 8); borda(b, Color3.fromRGB(255,80,80), 1.5, 0.4)
    return b
end

local function toggle(pai, txt, y, inicial, cb)
    local row = novo("TextButton", {Size = UDim2.new(1,-12,0,34), Position = UDim2.new(0,6,0,y),
        BackgroundColor3 = FUNDO2, Text = "", AutoButtonColor = false}, pai)
    canto(row, 8); borda(row, VERM, 1, 0.5)
    texto(row, txt, UDim2.new(0,10,0,0), UDim2.new(1,-62,1,0), Enum.Font.GothamBold, 12)
    local pill = novo("Frame", {Size = UDim2.new(0,42,0,20), Position = UDim2.new(1,-50,0.5,-10),
        BackgroundColor3 = CINZA_E}, row)
    canto(pill, 10)
    local knob = novo("Frame", {Size = UDim2.new(0,16,0,16), Position = UDim2.new(0,2,0.5,-8),
        BackgroundColor3 = Color3.new(1,1,1)}, pill)
    canto(knob, 8)
    local estado = false
    local function set(v)
        estado = v
        tween(pill, 0.15, {BackgroundColor3 = v and VERDE or CINZA_E})
        tween(knob, 0.15, {Position = v and UDim2.new(1,-18,0.5,-8) or UDim2.new(0,2,0.5,-8)})
        if cb then cb(v) end
    end
    row.MouseButton1Click:Connect(function() set(not estado) end)
    if inicial then set(true) end
    return {set = set, get = function() return estado end}
end

-- ---------- página Compra ----------
local pCompra = pagina("Compra")
toggle(pCompra, "Auto Compra", 4, false, function(v)
    ativo = v
    if v then task.spawn(loopCompra) else status("parado") end
end)
toggle(pCompra, "Usar Pedir", 42, USAR_PEDIR, function(v) usarPedir = v end)
local btnVel = botao(pCompra, "Velocidade: " .. MODOS[modoIdx].nome, 80)
btnVel.MouseButton1Click:Connect(function()
    modoIdx = modoIdx % #MODOS + 1
    btnVel.Text = "Velocidade: " .. MODOS[modoIdx].nome
end)
lblStatus = texto(pCompra, "status: parado", UDim2.new(0,8,0,118), UDim2.new(1,-12,0,18), Enum.Font.Gotham, 11, CINZA)
lblCount = texto(pCompra, "compras ok: 0", UDim2.new(0,8,0,136), UDim2.new(1,-12,0,18), Enum.Font.Gotham, 11, CINZA)
texto(pCompra, "anti-afk: ON", UDim2.new(0,8,0,154), UDim2.new(1,-12,0,18), Enum.Font.GothamBold, 11, VERDE)

-- ---------- página Jogador ----------
local pJog = pagina("Jogador")
local btnTP = botao(pJog, "TP pra Máquina", 6)
btnTP.MouseButton1Click:Connect(function()
    if tpMaquina(false) then status("na máquina", true) end
end)
texto(pJog, "Anti-AFK sempre ligado.", UDim2.new(0,8,0,48), UDim2.new(1,-12,0,18), Enum.Font.Gotham, 11, CINZA)

-- ---------- página Outros ----------
local pOut = pagina("Outros")
local btnRejoin = botao(pOut, "Reentrar no jogo", 6)
btnRejoin.MouseButton1Click:Connect(function()
    TPS:Teleport(game.PlaceId, player)
end)

-- ---------- painel Finder ----------
local finder = novo("Frame", {Position = UDim2.new(0,332,0,8), Size = UDim2.new(0,262,0,230),
    BackgroundColor3 = FUNDO, BorderSizePixel = 0, ClipsDescendants = true}, gui)
canto(finder, 10); borda(finder, LARANJA, 2, 0)
local finderScale = Instance.new("UIScale", finder); finderScale.Scale = 0

local fHeader = novo("Frame", {Size = UDim2.new(1,0,0,36), BackgroundColor3 = LARANJA, BorderSizePixel = 0}, finder)
gradiente(fHeader, Color3.fromRGB(255,180,60), Color3.fromRGB(230,110,20))
texto(fHeader, "Pack Finder", UDim2.new(0,10,0,0), UDim2.new(1,-100,1,0), Enum.Font.GothamBlack, 18)
local btnAutoF = novo("TextButton", {Size = UDim2.new(0,78,0,24), Position = UDim2.new(1,-86,0,6),
    BackgroundColor3 = Color3.fromRGB(50,50,58), Text = "AUTO OFF", TextColor3 = Color3.new(1,1,1),
    Font = Enum.Font.GothamBlack, TextSize = 12}, fHeader)
canto(btnAutoF, 6)
arrastar(fHeader, finder)

local lblRestock = texto(finder, "abra a Pack Machine pra ler o estoque", UDim2.new(0,10,0,38),
    UDim2.new(1,-20,0,18), Enum.Font.GothamBold, 11, CINZA)

local lista = novo("ScrollingFrame", {Position = UDim2.new(0,6,0,58), Size = UDim2.new(1,-12,1,-62),
    BackgroundTransparency = 1, BorderSizePixel = 0, ScrollBarThickness = 3,
    AutomaticCanvasSize = Enum.AutomaticSize.Y, CanvasSize = UDim2.new(0,0,0,0)}, finder)
local lay = Instance.new("UIListLayout"); lay.Padding = UDim.new(0,6); lay.Parent = lista

local cards = {}
for _, pack in ipairs(PACKS) do
    local card = novo("Frame", {Size = UDim2.new(1,-6,0,66), BackgroundColor3 = FUNDO2}, lista)
    canto(card, 8); borda(card, COR_PACK[pack] or VERM, 1.5, 0.3)
    local ico = novo("Frame", {Position = UDim2.new(0,6,0,10), Size = UDim2.new(0,46,0,46),
        BackgroundColor3 = COR_PACK[pack] or VERM}, card)
    canto(ico, 8)
    texto(ico, pack:sub(1,1), UDim2.new(0,0,0,0), UDim2.new(1,0,1,0), Enum.Font.GothamBlack, 24,
        Color3.new(1,1,1), Enum.TextXAlignment.Center)
    texto(card, "Pack " .. pack, UDim2.new(0,58,0,6), UDim2.new(1,-134,0,18), Enum.Font.GothamBlack, 14)
    local price = texto(card, PRECO[pack] .. "  •  " .. SORTE[pack] .. "x sorte", UDim2.new(0,58,0,24),
        UDim2.new(1,-134,0,16), Enum.Font.GothamBold, 11, Color3.fromRGB(110,230,130))
    local stock = texto(card, "estoque: ?", UDim2.new(0,58,0,42), UDim2.new(1,-134,0,16),
        Enum.Font.Gotham, 11, CINZA)
    local btn = novo("TextButton", {Size = UDim2.new(0,70,0,28), Position = UDim2.new(1,-76,0,19),
        BackgroundColor3 = LARANJA, Text = "Comprar", TextColor3 = Color3.new(1,1,1),
        Font = Enum.Font.GothamBlack, TextSize = 12}, card)
    canto(btn, 6)
    btn.MouseButton1Click:Connect(function()
        task.spawn(function()
            tpMaquina(true)
            for _ = 1, TENTATIVAS_FALHA do
                if comprarPack(pack) then break end
                task.wait(0.1)
            end
        end)
    end)
    cards[pack] = {stock = stock, price = price, btn = btn}
end

local function atualizarCards(dados, restock)
    lblRestock.Text = restock and ("Repõe em " .. restock) or "abra a Pack Machine pra ler o estoque"
    lblRestock.TextColor3 = restock and Color3.fromRGB(255,200,90) or CINZA
    for _, pack in ipairs(PACKS) do
        local c, d = cards[pack], dados[pack]
        if d then
            c.stock.Text = "x" .. d.stock .. " em estoque"
            c.stock.TextColor3 = d.stock > 0 and VERDE or CINZA
            c.btn.Text = d.stock > 0 and "Comprar" or "Sem estoque"
            c.btn.BackgroundColor3 = d.stock > 0 and LARANJA or CINZA_E
            c.price.Text = (d.price or PRECO[pack]) .. "  •  " .. (d.luck or SORTE[pack]) .. "x sorte"
        else
            c.stock.Text = "estoque: ?"
            c.stock.TextColor3 = CINZA
        end
    end
end

-- ---------- página Finder (controles) ----------
local pFind = pagina("Finder")
toggle(pFind, "Mostrar painel", 4, true, function(v) finder.Visible = v end)
local tgFinderAuto = toggle(pFind, "Auto no estoque", 42, false, function(v)
    finderAuto = v
    btnAutoF.Text = v and "AUTO ON" or "AUTO OFF"
    tween(btnAutoF, 0.15, {BackgroundColor3 = v and Color3.fromRGB(0,150,50) or Color3.fromRGB(50,50,58)})
end)
btnAutoF.MouseButton1Click:Connect(function() tgFinderAuto.set(not finderAuto) end)
texto(pFind, "Compra só os packs que", UDim2.new(0,8,0,82), UDim2.new(1,-12,0,16), Enum.Font.Gotham, 11, CINZA)
texto(pFind, "têm estoque agora.", UDim2.new(0,8,0,98), UDim2.new(1,-12,0,16), Enum.Font.Gotham, 11, CINZA)

-- ---------- minimizar / fechar ----------
local minimizado = false
btnMin.MouseButton1Click:Connect(function()
    minimizado = not minimizado
    btnMin.Text = minimizado and "+" or "–"
    tween(main, 0.3, {Size = UDim2.new(0,310,0, minimizado and 34 or 230)})
end)
btnX.MouseButton1Click:Connect(function()
    ativo = false; finderAuto = false
    tween(mainScale, 0.3, {Scale = 0}, Enum.EasingStyle.Back, Enum.EasingDirection.In)
    tween(finderScale, 0.3, {Scale = 0}, Enum.EasingStyle.Back, Enum.EasingDirection.In)
    task.wait(0.35)
    gui:Destroy()
end)

-- ---------- abrir ----------
mostrar("Compra")
tween(mainScale, 0.5, {Scale = 1}, Enum.EasingStyle.Back)
task.wait(0.15)
tween(finderScale, 0.5, {Scale = 1}, Enum.EasingStyle.Back)

-- ================= LOOP DO FINDER =================
task.spawn(function()
    while gui and gui.Parent do
        local dados, restock = lerCartas()
        atualizarCards(dados, restock)
        if finderAuto then
            for _, pack in ipairs(PACKS) do
                local d = dados[pack]
                if finderAuto and d and d.stock > 0 then
                    tpMaquina(true)
                    for _ = 1, d.stock do
                        if not finderAuto then break end
                        comprarPack(pack)
                        task.wait(MODOS[modoIdx].entre)
                    end
                end
            end
        end
        task.wait(0.5)
    end
end)# AutoPack.lua