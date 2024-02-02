// ==UserScript==
// @name         Inevitavel Script
// @namespace    http://tampermonkey.net/
// @version      0.1
// @description  try to take over the world!
// @author       You
// @match        https://www.habblet.city/hotelv2
// @icon         https://www.google.com/s2/favicons?sz=64&domain=habblet.city
// @grant        none
// ==/UserScript==

let hideTyping = false;
let copyUsers = false;
let muteAll = false;
let anonMode = false;
let translateUsers = false;
let users = [];
const colors = [
    'red',
    'purple',
    'orange',
    'yellow',
    'green',
    'cyan',
    'white',
    'blue'
];
let coloredMessages = false;
let lastSelection = 0;
let selectedColor = 0;
let lastSelectionA = 0;
let selectedLanguage = 0;

const caesarCipher = (str, shift, decrypt = false) => {
  const s = decrypt ? (26 - shift) % 26 : shift;
  const n = s > 0 ? s : 26 + (s % 26);
  return [...str]
    .map((l, i) => {
      const c = str.charCodeAt(i);
      if (c >= 65 && c <= 90)
        return String.fromCharCode(((c - 65 + n) % 26) + 65);
      if (c >= 97 && c <= 122)
        return String.fromCharCode(((c - 97 + n) % 26) + 97);
      return l;
    })
    .join('');
};

class RoomUnitEvent {

  static parse(packet) {
    for(let i = packet.readInt(); i > 0; i--) {
      const id = packet.readInt();
      const username = packet.readString();
      const motto = packet.readString();
      const figure = packet.readString();
      const roomIndex = packet.readInt();
      const x = packet.readInt();
      const y = packet.readInt();
      const z = parseFloat(packet.readString());
      const direction = packet.readInt();
      const type = packet.readInt();
      let unit = null;

      if(type == 1) {
          unit = {
              group: {},
              parse() {
                  this.gender = packet.readString();
                  this.group.id = packet.readInt();
                  this.group.status = packet.readInt();
                  this.group.name = packet.readString();
                  const _swimFigure = packet.readString();

                  this.achievementScore = packet.readInt();
                  this.isModerator = packet.readBoolean();
              }
          }
      } else if(type == 4) {
          unit = {
              skills: [],
              parse() {
                  this.gender = packet.readString();
                  this.ownerId = packet.readInt();
                  this.ownerName = packet.readString();
                  const totalSkills = packet.readInt();
                  for(let i = 0; i < totalSkills; i++) {
                      this.skills.push(packet.readShort());
                  }
              }
          }
      } else if (type == 2) {
          unit = {
              parse() {
                  this.subType = packet.readInt().toString();
                  this.ownerId = packet.readInt();
                  this.ownerName = packet.readString();
                  this.rarityLevel = packet.readInt();
                  this.hasSaddle = packet.readBoolean();
                  this.isRiding = packet.readBoolean();
                  this.canBreed = packet.readBoolean();
                  this.canHarvest = packet.readBoolean();
                  this.canRevive = packet.readBoolean();
                  this.hasBreedingPermission = packet.readBoolean();
                  this.petLevel = packet.readInt();
                  this.petPosture = packet.readString();
              }
          }
      } else {
        console.log(`Error! UnitType => ${type}`);
        return;
      }

      unit.id = id;
      unit.type = type;
      unit.username = username;
      unit.motto = motto;
      unit.figure = figure;
      unit.roomIndex = roomIndex;
      unit.x = x;
      unit.y = y;
      unit.z = z;
      unit.targetX = x;
      unit.targetY = y;
      unit.targetZ = z;
      unit.bodyDirection = direction;
      unit.headDirection = direction;

      unit.parse(packet);

      let _id;
      if(!users.find((user, i) => { return (_id = i, user.id == id); })) {
        users.push(unit);
      } else {
          users[_id] = unit;
      }
    }
  }
}
class BinaryReader {
    constructor(binary) {
        this.binary = binary;
        this.view = new DataView(binary);
        this.offset = 0;
    }
    readInt() {
        const value = this.view.getInt32(this.offset);
        this.offset += 4;
        return value;
    }
    readShort() {
        const value = this.view.getInt16(this.offset);
        this.offset += 2;
        return value;
    }
    readBoolean() {
        return !!this.binary[this.offset++];
    }
    readString() {
        const length = this.readShort();
        const str = new TextDecoder().decode(this.binary.slice(this.offset, this.offset + length));
        this.offset += length;
        return str;
    }
}
class BinaryWriter {
    constructor(header) {
        this.binary = [];
        this.offset = 0;
        this.writeInt(0);
        this.writeShort(header);
    }
    writeInt(value) {
        this.binary[this.offset++] = (value >> 24) & 0xFF;
        this.binary[this.offset++] = (value >> 16) & 0xFF;
        this.binary[this.offset++] = (value >> 8) & 0xFF;
        this.binary[this.offset++] = value & 0xFF;
        return this;
    }
    writeShort(value) {
        this.binary[this.offset++] = (value >> 8) & 0xFF;
        this.binary[this.offset++] = value & 0xFF;
        return this;
    }
    writeString(value) {
        const data = new TextEncoder().encode(value);
        this.writeShort(data.length);
        for(let i = 0; i < data.length; i++) {
            this.binary[this.offset + i] = data[i];
        }
        this.offset += data.length;
        return this;
    }
    compose() {
        this.offset = 0;
        this.writeInt(this.binary.length - 4);
        return new Uint8Array(this.binary).buffer;
    }
}

if (WebSocket.prototype.constructor.name != "WS") {
    const ws = window.WebSocket;
    class WS extends EventTarget {
        set onmessage(value) {
            this.ws.onmessage = value;
        }
        set onopen(value) {
            this.ws.onopen = value;
        }
        set onerror(value) {
            this.ws.onerror = value;
        }
        set onclose(value) {
            this.ws.onclose = value;
        }
        constructor(...args) {
            super();
            this.ws = new ws(...args);
            this.ws.binaryType = "arraybuffer";
            window.wss = this;

            this.ws.addEventListener("message", async (e) => {
                const reader = new BinaryReader(e.data);
                reader.readInt();
                const header = reader.readShort();
                if (header == 374) {
                    RoomUnitEvent.parse(reader);
                } else if (header == 1446) {
                    if (muteAll) {
                        return;
                    }

                    let roomIndex = reader.readInt();
                    let message = reader.readString();
                    let gesture = reader.readInt();
                    let bubble = reader.readInt();

                    if (anonMode && message.startsWith("ms!")) {
                        message = "[Anônimo] " + caesarCipher(message.substring(3), 13, true);
                        let length = new TextEncoder().encode(message).length;
                        const p = new BinaryWriter(1446);
                        p.writeInt(roomIndex);
                        p.writeString(message);
                        p.writeInt(gesture);
                        p.writeInt(bubble);
                        p.writeInt(0);
                        p.writeInt(length);
                        return this.dispatchEvent(
                            new MessageEvent("message", {
                                data: p.compose(),
                            })
                        );
                    } else if (translateUsers) {
                        const res = await fetch(`https://bb0b19cb-36d7-4c54-9e63-3e36600a3d65-00-37y3cctdon927.picard.replit.dev/?source=en&target=pt&query=${message}`, {
                            //mode: 'no-cors'
                        });
                        const json = await res.json();

                        let length = new TextEncoder().encode(json.translatedText).length;
                        const p = new BinaryWriter(1446);
                        p.writeInt(roomIndex);
                        p.writeString(json.translatedText);
                        p.writeInt(gesture);
                        p.writeInt(bubble);
                        p.writeInt(0);
                        p.writeInt(length);
                        return this.dispatchEvent(
                            new MessageEvent("message", {
                                data: p.compose(),
                            })
                        );
                    }
                }
                this.dispatchEvent(
                    new MessageEvent("message", {
                        data: e.data,
                    })
                );
            });

            this.ws.addEventListener("open", (e) => {
                this.dispatchEvent(
                    new MessageEvent("open", {
                        data: e.data,
                    })
                );
            });

            this.ws.addEventListener("close", (e) => {
                this.dispatchEvent(
                    new MessageEvent("close", {
                        data: e.data,
                    })
                );
            });

            this.ws.addEventListener("error", (e) => {
                this.dispatchEvent(
                    new MessageEvent("error", {
                        data: e.data,
                    })
                );
            });
        }
        send(data) {
            this.ws.send(data);
        }
    }

    window.WebSocket = WS;
}

function loadAll() {
    const script1 = document.createElement("script");
    script1.src = "https://flyover.github.io/imgui-js/dist/imgui.umd.js";

    const script2 = document.createElement("script");
    script2.src = "https://flyover.github.io/imgui-js/dist/imgui_impl.umd.js";


    script1.onload = function() {
        document.body.appendChild(script2);
    }

    script2.onload = function() {
        start();
    }


    document.body.appendChild(script1);
}

async function start() {
    const send = window.wss.send;
    window.wss.send = async function (data) {
        const reader = new BinaryReader(data);
        reader.readInt();
        const header = reader.readShort();
        if (header == 1314) {
            let message = reader.readString();
            let bubble = reader.readInt();
            let argv = message.split(" ");
            if (argv[0] == ":copy") {
                if (copyUsers) {
                    const user = users.find((user) => user.username == argv[1]);
                    if (!user) return;
                    const packet = new BinaryWriter(2730);
                    packet.writeString(user.gender);
                    packet.writeString(user.figure);
                    data = packet.compose();
                }
            } else if (coloredMessages) {
                const packet = new BinaryWriter(1314);
                packet.writeString(`@${colors[selectedColor]}@${message}`);
                packet.writeInt(bubble);
                send.bind(window.wss)(packet.compose());
                return;
            } else if (anonMode) {
                const packet = new BinaryWriter(1314);
                const msg = message.length >= 100 ? message.substring(0, message.length - 3) : message;
                packet.writeString(`ms!${caesarCipher(msg, 13)}`);
                packet.writeInt(bubble);
                send.bind(window.wss)(packet.compose());
                return;
            } else if (translateUsers) {
                const res = await fetch(`https://bb0b19cb-36d7-4c54-9e63-3e36600a3d65-00-37y3cctdon927.picard.replit.dev/?source=pt&target=en&query=${message}`, {
                    //mode: 'no-cors'
                });
                const json = await res.json();

                const packet = new BinaryWriter(1314);
                packet.writeString(json.translatedText);
                packet.writeInt(bubble);
                send.bind(window.wss)(packet.compose());
                return;
            }
        } else if (header == 1597) {
            if (hideTyping) {
                return;
            }
        }
        send.bind(window.wss)(data);
    };

    const gamecvs = document.querySelector("canvas");
    let isMouseHoveringGui = false;
    let isRendering = true;

    const canvas = document.createElement("canvas");
    canvas.width = innerWidth;
    canvas.height = innerHeight;
    canvas.style.top = "0px";
    canvas.style.left = "0px";
    canvas.style.position = "absolute";
    canvas.style.zIndex = "21";

    window.addEventListener("resize", (e) => {
        canvas.width = innerWidth;
        canvas.height = innerHeight;
    });

    canvas.addEventListener("mousemove", (e) => {
        if (isMouseHoveringGui) return;

        var newEvent = new MouseEvent("mousemove", {
            bubbles: true,
            cancelable: true,
            clientX: e.clientX,
            clientY: e.clientY,
        });

        gamecvs.dispatchEvent(newEvent);
    });

    canvas.addEventListener("click", (e) => {
        if (isMouseHoveringGui) return;

        var newEvent = new MouseEvent("click", {
            bubbles: true,
            cancelable: true,
            clientX: e.clientX,
            clientY: e.clientY,
        });

        gamecvs.dispatchEvent(newEvent);
    });

    canvas.addEventListener("mousedown", (e) => {
        if (isMouseHoveringGui) return;

        var newEvent = new MouseEvent("mousedown", {
            bubbles: true,
            cancelable: true,
            clientX: e.clientX,
            clientY: e.clientY,
        });

        gamecvs.dispatchEvent(newEvent);
    });

    canvas.addEventListener("mouseup", (e) => {
        if (isMouseHoveringGui) return;

        var newEvent = new MouseEvent("mouseup", {
            bubbles: true,
            cancelable: true,
            clientX: e.clientX,
            clientY: e.clientY,
        });

        gamecvs.dispatchEvent(newEvent);
    });

    document.addEventListener("keydown", (e) => {
        if (e.key == "Escape") isRendering = !isRendering;
    });

    document.body.appendChild(canvas);

    const glContext = canvas.getContext("webgl", { alpha: true });

    await ImGui.default();
    ImGui.CreateContext();
    ImGui_Impl.Init(canvas);

    ImGui.StyleColorsDark();

    let done = false;
    window.requestAnimationFrame(_loop);
    function _loop(time) {
        ImGui_Impl.NewFrame(time);
        ImGui.NewFrame();
        ImGui.SetNextWindowPos(new ImGui.ImVec2(20, 20), ImGui.Cond.FirstUseEver);
        ImGui.SetNextWindowSize(
            new ImGui.ImVec2(294, 140),
            ImGui.Cond.FirstUseEver
        );

        if (isRendering) {
            ImGui.Begin("Utils");
            ImGui.Checkbox("Esconder balão digitando", (value = hideTyping) => hideTyping = value);
            ImGui.SameLine();
            ImGui.TextDisabled("(?)");
            if (ImGui.IsItemHovered()) {
                ImGui.BeginTooltip();
                ImGui.PushTextWrapPos(ImGui.GetFontSize() * 35.0);
                ImGui.TextUnformatted("Ativando esta função, ao digitar uma mensagem, não irá mostrar o balão de digitando.");
                ImGui.PopTextWrapPos();
                ImGui.EndTooltip();
            }
            ImGui.Checkbox("Copiar visuais", (value = copyUsers) => copyUsers = value);
            ImGui.SameLine();
            ImGui.TextDisabled("(?)");
            if (ImGui.IsItemHovered()) {
                ImGui.BeginTooltip();
                ImGui.PushTextWrapPos(ImGui.GetFontSize() * 35.0);
                ImGui.TextUnformatted("Ativando esta função, você poderá usar o comando ':copy' dos VIP's.");
                ImGui.PopTextWrapPos();
                ImGui.EndTooltip();
            }
            ImGui.Checkbox("Mutar todos da sala", (value = muteAll) => muteAll = value);
            ImGui.SameLine();
            ImGui.TextDisabled("(?)");
            if (ImGui.IsItemHovered()) {
                ImGui.BeginTooltip();
                ImGui.PushTextWrapPos(ImGui.GetFontSize() * 35.0);
                ImGui.TextUnformatted("Ativando esta função, você não irá ver mais as mensagens da sala (só você).");
                ImGui.PopTextWrapPos();
                ImGui.EndTooltip();
            }
            if(ImGui.Button("Kikar todos")) {
                for(const user of users) {
                    const writer = new BinaryWriter(1320);
                    writer.writeInt(user.id);
                    window.wss.send(writer.compose());
                }
            }
            ImGui.SameLine();
            ImGui.TextDisabled("(?)");
            if (ImGui.IsItemHovered()) {
                ImGui.BeginTooltip();
                ImGui.PushTextWrapPos(ImGui.GetFontSize() * 35.0);
                ImGui.TextUnformatted("Ao clicar neste botão, você irá kikar todos da sala (você precisa ter direitos na sala).");
                ImGui.PopTextWrapPos();
                ImGui.EndTooltip();
            }

            ImGui.Checkbox('Cor da mensagem', (value = coloredMessages) => coloredMessages = value);
            ImGui.SameLine();
            if(ImGui.Combo('##Selector', (value = lastSelection) => (lastSelection = value, selectedColor), colors.join('\0'))) {
                selectedColor = lastSelection;
            }
            ImGui.Checkbox('Modo anônimo', (value = anonMode) => anonMode = value);
            ImGui.Checkbox('Traduzir jogadores', (value = translateUsers) => translateUsers = value);

            ImGui.Text("Version 1.4");

            const mousePos = ImGui.GetMousePos();
            const windowPos = ImGui.GetWindowPos();
            const windowSize = ImGui.GetWindowSize();

            isMouseHoveringGui =
                mousePos.x >= windowPos.x &&
                mousePos.x <= windowPos.x + windowSize.x &&
                mousePos.y >= windowPos.y &&
                mousePos.y <= windowPos.y + windowSize.y;

            ImGui.End();
        } else {
            isMouseHoveringGui = false;
        }

        const friendRequests = document.querySelector(".position-absolute.visible.object-location");

        canvas.style.zIndex = isMouseHoveringGui ? "21" : "0";
        if(friendRequests) friendRequests.style.zIndex = isMouseHoveringGui ? "0" : "1";

        ImGui.EndFrame();

        ImGui.Render();

        const gl = glContext;
        gl.viewport(0, 0, gl.drawingBufferWidth, gl.drawingBufferHeight);
        gl.colorMask(true, true, true, true);
        gl.clear(gl.COLOR_BUFFER_BIT);
        ImGui_Impl.RenderDrawData(ImGui.GetDrawData());

        window.requestAnimationFrame(done ? _done : _loop);
    }
}

function _done() {
    ImGui_Impl.Shutdown();
    ImGui.DestroyContext();
}

let loadStarted = false;
const getContext = HTMLCanvasElement.prototype.getContext;
HTMLCanvasElement.prototype.getContext = function(...args) {
    const ctx = getContext.call(this, ...args);
    if (args[0] == "webgl2" && !loadStarted) {
        loadStarted = true;
        loadAll();
    }
    return ctx;
}
