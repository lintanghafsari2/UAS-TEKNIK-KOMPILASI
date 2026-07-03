# UAS-TEKNIK-KOMPILASI
Uas
UAS Teknik Kompilasi - Lintang Nur Hafsari

UNIVERSITAS PAMULANG
Ujian Akhir Semester (UAS) – Teknik Kompilasi



SIMULASI PROSES KOMPILASI KONSTRUKSI DEKLARASI FUNGSI
Tahapan: Analisis Leksikal → Sintaksis → Semantik → Generasi Kode Antara (Three-Address Code)




Disusun oleh: Lintang Nur Hafsari NIM : 231011400071 Kelas : 06TPLE002



Mata Kuliah
Teknik Kompilasi





2026

Dokumentasi: Simulasi Proses Kompilasi Konstruksi Deklarasi Fungsi
Nama : Lintang Nur Hafsari NIM : 231011400071 Kelas : 06TPLE002 Tugas : UAS Teknik Kompilasi

Konstruksi yang Dipilih
Konstruksi sintaksis yang diimplementasikan adalah deklarasi fungsi/metode. Konstruksi ini dipilih karena memperkenalkan elemen yang tidak muncul pada konstruksi kontrol alur biasa (if-else, while), yaitu:
Daftar parameter yang harus divalidasi tidak boleh duplikat,
Scope (ruang lingkup) lokal milik fungsi, tempat parameter dan variabel lokal hidup,
Tipe kembalian (return type) yang harus dicocokkan dengan tipe ekspresi pada setiap statement return.
Bahasa mini yang dirancang mendukung satu deklarasi fungsi dengan: - Parameter bertipe int atau float - Deklarasi variabel lokal - Assignment dengan ekspresi aritmatika (+ - * /, tanda kurung) - Statement return <ekspresi>;
Contoh program input:


Pattern / Grammar (BNF)
<program>	::= <func_decl>
<func_decl>	::= TYPE IDENT "(" <param_list> ")" "{" <stmt_list> "}"
<param_list> ::= [ <param> { "," <param> } ]
<param>	::= TYPE IDENT

<stmt_list>	::= { <stmt> }
<stmt>	::= <decl> | <assign> | <return_stmt>

<decl>	::= TYPE IDENT ";"
<assign>	::= IDENT "=" <expr> ";"
<return_stmt> ::= "return" <expr> ";"

<expr>	::= <term> { ("+" | "-") <term> }
<term>	::= <factor> { ("*" | "/") <factor> }
<factor>	::= IDENT | NUMBER | "(" <expr> ")"

Grammar bersifat LL(1). Aturan <param_list> memakai pilihan opsional ([ ... ]) untuk menangani fungsi tanpa parameter, dan pengulangan { "," <param> } untuk mendukung jumlah parameter yang tidak terbatas — pola yang identik dengan penanganan daftar argumen pada kebanyakan bahasa pemrograman nyata.

Struktur Implementasi Program
Program ditulis dalam Python 3 (compiler_simulation_function.py) dengan gaya berorientasi objek yang memanfaatkan dataclass dan Enum dari pustaka standar Python — sedikit berbeda dari pendekatan fungsional yang dipakai pada simulasi konstruksi lain, untuk menekankan representasi AST sebagai struktur data yang eksplisit dan mudah diperiksa tipenya.

Tahap 1 — Analisis Leksikal (Lexer)

TokenKind adalah Enum yang mendaftarkan seluruh kategori token (NUMBER, TYPE, RETURN, IDENT, RELOP, dst).
Kelas Lexer membungkus proses tokenize(): memakai satu regex gabungan (_MASTER) dari daftar

_TOKEN_PATTERNS, lalu memetakan nama grup regex ke anggota TokenKind yang sesuai lewat kind_map.
Kata kunci int/float langsung dikenali sebagai token TYPE oleh pola regex \b(?:int|float)\b, tanpa perlu tahap klasifikasi ulang seperti pada simulasi lain — pendekatan yang berbeda namun setara.
Baris (line) dilacak untuk pelaporan error yang presisi.
Output tahap ini: List[Token], masing-masing sebagai @dataclass berisi kind, lexeme, dan line.

Tahap 2 — Analisis Sintaksis (Parser → AST)

AST direpresentasikan sebagai kumpulan @dataclass: FuncDecl, Param, VarDecl, Assign, Return, BinExpr, Ident, Literal. Penggunaan dataclass membuat setiap node otomatis punya representasi string dan perbandingan nilai, memudahkan proses debugging AST.
Parser adalah recursive-descent parser klasik: method _match() memverifikasi & mengonsumsi token, melempar [SINTAKSIS] error jika tidak sesuai grammar.
Karena grammar mendukung daftar parameter, method _param_list() secara eksplisit menangani kasus fungsi tanpa parameter (RPAREN langsung ditemukan) maupun banyak parameter dipisah koma.
print_tree() mencetak AST fungsi lengkap dengan signature (FuncDecl(int hitungLuas(int panjang, int lebar))), termasuk seluruh isi body secara berindentasi.

Tahap 3 — Analisis Semantik (SemanticAnalyzer)
Karena konstruksi ini memiliki scope, SemanticAnalyzer membangun satu tabel scope (self.scope) yang diisi dua kali:
Parameter didaftarkan lebih dulu ke scope (p.name -> p.ptype); parameter dengan nama duplikat langsung dilaporkan sebagai error.
Body fungsi diproses statement demi statement:
 VarDecl yang namanya sudah ada di scope (baik karena bentrok dengan parameter maupun deklarasi lokal lain) ditolak.
 Assign divalidasi: variabel harus ada di scope, dan tipe hasil ekspresi (_type_of) harus kompatibel dengan tipe variabel tujuan (aturan implicit widening: int <- int valid, float <-int atau float <- float valid, int <- float ditolak).
 Return divalidasi terhadap tipe kembalian fungsi: jika fungsi dideklarasikan int tetapi ekspresi return-nya bertipe float, dilaporkan sebagai error semantik.
Sebagai pengecekan tambahan yang khas untuk fungsi, analyzer juga memverifikasi bahwa fungsi memiliki minimal satu statement return; ketiadaannya dilaporkan sebagai error.
Seluruh error dikumpulkan dalam self.errors dan dilaporkan sekaligus melalui exception SemanticError.

Tahap 4 — Generasi Kode Antara (CodeGenerator)
Skema TAC untuk deklarasi fungsi mengikuti pola umum kompilator nyata, dengan penanda awal/akhir fungsi secara eksplisit:
FUNC_BEGIN nama_fungsi
; param <tipe> <nama>	(satu baris komentar per parameter)
<kode body>
RETURN <tempat_nilai> FUNC_END nama_fungsi

Parameter tidak menghasilkan instruksi TAC yang dieksekusi (nilainya dianggap sudah terikat oleh mekanisme pemanggilan/calling convention di luar cakupan simulasi ini), hanya dicatat sebagai komentar dokumentasi.
Ekspresi biner (_gen_expr) menghasilkan variabel sementara baru (t1, t2, ...) melalui
_new_temp(), sama seperti pola umum TAC pada ekspresi aritmatika.
Statement return diterjemahkan menjadi instruksi RETURN <place>, yang secara konseptual setara dengan menyalin nilai ke register/alamat hasil sebelum kendali program kembali ke pemanggil.
Penanda FUNC_BEGIN/FUNC_END membedakan blok kode ini dari TAC konstruksi lain (seperti if-else
atau while) yang tidak memiliki batas awal/akhir prosedur secara eksplisit.

Contoh Eksekusi Program
Kasus Valid

Input:

AST (ringkas):

FuncDecl(int hitungLuas(int panjang, int lebar)) VarDecl(int luas)
Assign(luas =) -> BinExpr(*) -> Ident(panjang), Ident(lebar) Return -> Ident(luas)

Hasil Analisis Semantik:

Status: LULUS. Tidak ditemukan kesalahan semantik. Scope Fungsi: panjang : int, lebar : int, luas : int

Three-Address Code yang Dihasilkan:

FUNC_BEGIN hitungLuas
; param int panjang
; param int lebar
; local int luas
t1 = panjang * lebar luas = t1
RETURN luas FUNC_END hitungLuas

Kasus Error Semantik
Input:

Program lolos tahap Leksikal dan Sintaksis (grammar tetap valid karena float sisi; adalah bentuk
<decl> yang sah secara sintaksis), tetapi gagal di tahap Semantik dengan dua error:

[SEMANTIK] Baris 3: variabel 'sisi' sudah dideklarasikan (bentrok dengan deklarasi/parameter sebelumnya).
[SEMANTIK] Baris 4: mengembalikan nilai 'float' dari fungsi bertipe kembalian 'int'.
Error pertama menunjukkan bahwa nama parameter sisi tidak boleh dideklarasikan ulang sebagai variabel lokal — sesuatu yang mustahil dideteksi tanpa konsep scope, yang justru menjadi ciri khas pengecekan semantik pada konstruksi fungsi. Error kedua menunjukkan pengecekan tipe kembalian terhadap tipe fungsi. Karena tahap semantik gagal, proses tidak dilanjutkan ke tahap Generasi Kode Antara.

Kesimpulan
Implementasi ini menunjukkan bagaimana konstruksi deklarasi fungsi diproses melalui empat tahap kompilasi yang saling bergantung:

Tahap	Input	Output	Tujuan
Leksikal	String source code	List of Token	Mengenali unit terkecil
Memverifikasi &

Sintaksis	List of Token	AST (FuncDecl)


Semantik	AST	Scope Fungsi / Error
merepresentasikan signature + body fungsi
Memvalidasi parameter, deklarasi, dan tipe kembalian

Generasi Kode	AST tervalidasi	Three-Address Code	Kode antara dengan
penanda
FUNC_BEGIN/FUNC_END

Dibandingkan konstruksi if-else maupun while, ciri khas deklarasi fungsi terletak pada tahap semantik: dibutuhkan konsep scope untuk menampung parameter sekaligus variabel lokal, dan diperlukan pengecekan tambahan berupa kecocokan tipe kembalian. Pada tahap generasi kode, hasil TAC dibungkus oleh penanda awal/akhir prosedur (FUNC_BEGIN / FUNC_END) yang tidak muncul pada TAC konstruksi kontrol alur biasa.

File yang Disertakan
compiler_simulation_function.py — kode program lengkap (dapat dijalankan dengan python3 compiler_simulation_function.py), sudah menyertakan dua contoh program (valid dan error) yang otomatis dijalankan melalui seluruh pipeline kompilasi.
Dokumen ini (versi PDF) yang menjelaskan grammar, arsitektur kode, dan hasil eksekusi program.

Lampiran: Kode Sumber Lengkap (compiler_simulation_function.py)
"""
===================================================================== SIMULASI PROSES KOMPILASI KONSTRUKSI: DEKLARASI FUNGSI (FUNCTION DECL)
=====================================================================
Nama	: Lintang Nur Hafsari NIM	: 231011400071
Kelas : 06TPLE002
Tugas : UAS Teknik Kompilasi

Tahapan yang disimulasikan:
Analisis Leksikal	-> memecah source code menjadi token
Analisis Sintaksis	-> membangun Abstract Syntax Tree (AST)
Analisis Semantik	-> pengecekan parameter, deklarasi & tipe
Generasi Kode Antara -> menghasilkan Three-Address Code (TAC) Bahasa mini yang didukung (lihat dokumentasi untuk grammar BNF lengkap):
int hitungLuas(int panjang, int lebar) { int luas;
luas = panjang * lebar; return luas;
}

Berbeda dari simulasi if-else maupun while-loop, konstruksi ini berfokus pada aspek DEKLARASI FUNGSI: daftar parameter, ruang lingkup (scope) lokal fungsi, serta kesesuaian tipe nilai balik (return type checking). Implementasi ditulis dengan gaya berorientasi objek memakai `dataclass` sebagai representasi node AST.
===================================================================== """

from dataclasses import dataclass, field from enum import Enum, auto
from typing import List, Optional import re


# ===================================================================== # TAHAP 1: ANALISIS LEKSIKAL
# =====================================================================

class TokenKind(Enum): NUMBER = auto()
TYPE = auto()	# int / float
RETURN = auto() IDENT = auto() COMMA = auto() ASSIGN = auto() RELOP = auto() PLUS = auto() MINUS = auto() STAR = auto() SLASH = auto() LPAREN = auto() RPAREN = auto() LBRACE = auto() RBRACE = auto() SEMI = auto() EOF = auto()


@dataclass
class Token:
kind: TokenKind lexeme: str line: int

_TOKEN_PATTERNS = [
("NUMBER", r"\d+(\.\d+)?"),
("TYPE",	r"\b(?:int|float)\b"), ("RETURN", r"\breturn\b"),
("IDENT",	r"[A-Za-z_][A-Za-z0-9_]*"), ("RELOP",	r"==|!=|<=|>=|<|>"),
("ASSIGN", r"="),
("COMMA",	r","),
("PLUS",	r"\+"),
("MINUS",	r"-"),
("STAR",	r"\*"),
("SLASH",	r"/"),
("LPAREN", r"\("),
("RPAREN", r"\)"),
("LBRACE", r"\{"),
("RBRACE", r"\}"),
("SEMI",	r";"),
("NEWLINE", r"\n"),
("SPACE",	r"[ \t]+"),
("BAD",	r"."),
]

_MASTER = "|".join(f"(?P<{n}>{p})" for n, p in _TOKEN_PATTERNS)


class Lexer:
"""Kelas pembungkus proses tokenisasi (Analisis Leksikal)."""

def  init (self, source: str): self.source = source

def tokenize(self) -> List[Token]: tokens: List[Token] = []
line = 1
for m in re.finditer(_MASTER, self.source): group = m.lastgroup
text = m.group()

if group == "NEWLINE": line += 1 continue
if group == "SPACE":
continue
if group == "BAD":
raise SyntaxError(f"[LEKSIKAL] Baris {line}: karakter tak dikenal {text!r}")

kind_map = {
"NUMBER": TokenKind.NUMBER,
"TYPE": TokenKind.TYPE, "RETURN": TokenKind.RETURN, "IDENT": TokenKind.IDENT, "RELOP": TokenKind.RELOP, "ASSIGN": TokenKind.ASSIGN, "COMMA": TokenKind.COMMA,
"PLUS": TokenKind.PLUS, "MINUS": TokenKind.MINUS,
"STAR": TokenKind.STAR, "SLASH": TokenKind.SLASH, "LPAREN": TokenKind.LPAREN, "RPAREN": TokenKind.RPAREN, "LBRACE": TokenKind.LBRACE, "RBRACE": TokenKind.RBRACE,
"SEMI": TokenKind.SEMI,
}
tokens.append(Token(kind_map[group], text, line))

tokens.append(Token(TokenKind.EOF, "", line)) return tokens


# ===================================================================== # TAHAP 2: ANALISIS SINTAKSIS -> ABSTRACT SYNTAX TREE
# =====================================================================
# Grammar BNF (recursive-descent, LL(1)):


#


#
<program>
::=
<func_decl>
#
<func_decl>
::=
TYPE IDENT "(" <param_list> ")" "{" <stmt_list> "}"
#
<param_list>
::=
[ <param> { "," <param> } ]
#
<param>
::=
TYPE IDENT
#






#
<stmt_list>
::=
{ <stmt> }
#
<stmt>
::=
<decl> | <assign> | <return_stmt>
#






#
<decl>
::=
TYPE IDENT ";"
#
<assign>
::=
IDENT "=" <expr> ";"
#
<return_stmt>
::=
"return" <expr> ";"
#






#
<expr>
::=
<term> { ("+" | "-") <term> }
#
<term>
::=
<factor> { ("*" | "/") <factor> }
#
<factor>
::=
IDENT | NUMBER | "(" <expr> ")"
#
=====================================================================


@dataclass
class Param:
ptype: str name: str line: int


@dataclass
class FuncDecl: return_type: str name: str
params: List[Param] body: List["Stmt"] line: int


@dataclass
class VarDecl: vtype: str name: str line: int


@dataclass
class Assign:
name: str expr: "Expr" line: int


@dataclass
class Return:
expr: "Expr" line: int


@dataclass
class BinExpr: left: "Expr" op: str right: "Expr" line: int


@dataclass
class Ident:
name: str line: int


@dataclass
class Literal: value: object line: int

Stmt = object	# alias informatif: VarDecl | Assign | Return
Expr = object	# alias informatif: BinExpr | Ident | Literal


class Parser:
"""Recursive-descent parser: token stream -> AST."""

def  init (self, tokens: List[Token]): self.tokens = tokens
self.pos = 0

def _peek(self) -> Token:
return self.tokens[self.pos]

def _advance(self) -> Token: tok = self.tokens[self.pos] self.pos += 1
return tok

def _match(self, kind: TokenKind) -> Token: tok = self._peek()
if tok.kind != kind:
raise SyntaxError(
f"[SINTAKSIS] Baris {tok.line}: diharapkan {kind.name}, " f"ditemukan {tok.kind.name} ({tok.lexeme!r})"
)
return self._advance()

# ---- entry point ----
def parse(self) -> FuncDecl: func = self._func_decl() self._match(TokenKind.EOF) return func

def _func_decl(self) -> FuncDecl: type_tok = self._match(TokenKind.TYPE)
name_tok = self._match(TokenKind.IDENT) self._match(TokenKind.LPAREN)
params = self._param_list() self._match(TokenKind.RPAREN) self._match(TokenKind.LBRACE) body = self._stmt_list() self._match(TokenKind.RBRACE)
return FuncDecl(type_tok.lexeme, name_tok.lexeme, params, body, type_tok.line)

def _param_list(self) -> List[Param]: params = []
if self._peek().kind == TokenKind.RPAREN:
return params params.append(self._param())
while self._peek().kind == TokenKind.COMMA: self._advance() params.append(self._param())
return params

def _param(self) -> Param:
type_tok = self._match(TokenKind.TYPE) name_tok = self._match(TokenKind.IDENT)
return Param(type_tok.lexeme, name_tok.lexeme, type_tok.line)

def _stmt_list(self) -> List[Stmt]: stmts = []
while self._peek().kind not in (TokenKind.RBRACE, TokenKind.EOF): stmts.append(self._stmt())
return stmts

def _stmt(self) -> Stmt: tok = self._peek()
if tok.kind == TokenKind.TYPE:
return self._var_decl()
if tok.kind == TokenKind.IDENT:
return self._assign()
if tok.kind == TokenKind.RETURN:
return self._return_stmt()

raise SyntaxError(f"[SINTAKSIS] Baris {tok.line}: statement tidak valid ({tok.kind.name})")

def _var_decl(self) -> VarDecl:
type_tok = self._match(TokenKind.TYPE) name_tok = self._match(TokenKind.IDENT) self._match(TokenKind.SEMI)
return VarDecl(type_tok.lexeme, name_tok.lexeme, type_tok.line)

def _assign(self) -> Assign:
name_tok = self._match(TokenKind.IDENT) self._match(TokenKind.ASSIGN)
expr = self._expr() self._match(TokenKind.SEMI)
return Assign(name_tok.lexeme, expr, name_tok.line)

def _return_stmt(self) -> Return:
ret_tok = self._match(TokenKind.RETURN) expr = self._expr() self._match(TokenKind.SEMI)
return Return(expr, ret_tok.line)

def _expr(self) -> Expr: node = self._term()
while self._peek().kind in (TokenKind.PLUS, TokenKind.MINUS): op_tok = self._advance()
right = self._term()
node = BinExpr(node, op_tok.lexeme, right, op_tok.line) return node

def _term(self) -> Expr: node = self._factor()
while self._peek().kind in (TokenKind.STAR, TokenKind.SLASH): op_tok = self._advance()
right = self._factor()
node = BinExpr(node, op_tok.lexeme, right, op_tok.line) return node

def _factor(self) -> Expr: tok = self._peek()
if tok.kind == TokenKind.NUMBER: self._advance()
value = float(tok.lexeme) if "." in tok.lexeme else int(tok.lexeme) return Literal(value, tok.line)
if tok.kind == TokenKind.IDENT: self._advance()
return Ident(tok.lexeme, tok.line) if tok.kind == TokenKind.LPAREN:
self._advance() node = self._expr()
self._match(TokenKind.RPAREN)
return node
raise SyntaxError(f"[SINTAKSIS] Baris {tok.line}: ekspresi tidak valid ({tok.kind.name})")


def print_tree(node, depth: int = 0) -> None: """Menampilkan AST fungsi sebagai pohon berindentasi.""" pad = " " * depth
if isinstance(node, FuncDecl):
params_str = ", ".join(f"{p.ptype} {p.name}" for p in node.params) print(f"{pad}FuncDecl({node.return_type} {node.name}({params_str}))") for s in node.body:
print_tree(s, depth + 1)
elif isinstance(node, VarDecl): print(f"{pad}VarDecl({node.vtype} {node.name})")
elif isinstance(node, Assign): print(f"{pad}Assign({node.name} =)") print_tree(node.expr, depth + 1)
elif isinstance(node, Return): print(f"{pad}Return") print_tree(node.expr, depth + 1)
elif isinstance(node, BinExpr): print(f"{pad}BinExpr({node.op})") print_tree(node.left, depth + 1) print_tree(node.right, depth + 1)

elif isinstance(node, Ident): print(f"{pad}Ident({node.name})")
elif isinstance(node, Literal): print(f"{pad}Literal({node.value})")


# ===================================================================== # TAHAP 3: ANALISIS SEMANTIK
# =====================================================================
# Pengecekan yang dilakukan:
#	a. Parameter tidak boleh memiliki nama yang duplikat #	b. Variabel lokal tidak boleh dideklarasikan dua kali #		(termasuk bentrok dengan nama parameter)
#	c. Setiap identifier yang dipakai (assign / return / ekspresi)
#		harus sudah ada di scope fungsi (parameter atau deklarasi lokal) #	d. Tipe hasil assignment harus kompatibel dengan tipe variabel
#		(int <- int OK, float <- int/float OK, int <- float DITOLAK) #	e. Tipe ekspresi pada `return` harus kompatibel dengan tipe
#	kembalian (return type) yang dideklarasikan fungsi
# =====================================================================

class SemanticError(Exception):
pass


class SemanticAnalyzer:
def   init  (self):
self.scope: dict = {}	# nama -> tipe (scope lokal fungsi)
self.errors: List[str] = []

def analyze(self, func: FuncDecl) -> dict:
# 1) daftarkan parameter ke scope for p in func.params:
if p.name in self.scope: self.errors.append(
f"[SEMANTIK] Baris {p.line}: parameter '{p.name}' duplikat."
)
else:
self.scope[p.name] = p.ptype

# 2) proses body fungsi has_return = False
for stmt in func.body:
if isinstance(stmt, Return): has_return = True
self._visit_stmt(stmt, func.return_type)

if not has_return: self.errors.append(
f"[SEMANTIK] Baris {func.line}: fungsi '{func.name}' tidak memiliki " f"statement 'return'."
)

if self.errors:
raise SemanticError("\n".join(self.errors))
return self.scope

def _visit_stmt(self, node, return_type: str):
if isinstance(node, VarDecl):
if node.name in self.scope: self.errors.append(
f"[SEMANTIK] Baris {node.line}: variabel '{node.name}' sudah " f"dideklarasikan (bentrok dengan deklarasi/parameter sebelumnya)."
)
else:
self.scope[node.name] = node.vtype

elif isinstance(node, Assign):
if node.name not in self.scope: self.errors.append(
f"[SEMANTIK] Baris {node.line}: variabel '{node.name}' belum " f"dideklarasikan."
)
return

rhs_type = self._type_of(node.expr) lhs_type = self.scope[node.name]
if lhs_type == "int" and rhs_type == "float": self.errors.append(
f"[SEMANTIK] Baris {node.line}: nilai 'float' tidak dapat " f"disimpan ke variabel '{node.name}' bertipe 'int'."
)

elif isinstance(node, Return): expr_type = self._type_of(node.expr)
if return_type == "int" and expr_type == "float": self.errors.append(
f"[SEMANTIK] Baris {node.line}: mengembalikan nilai 'float' " f"dari fungsi bertipe kembalian 'int'."
)

def _type_of(self, node) -> str:
if isinstance(node, Literal):
return "float" if isinstance(node.value, float) else "int" if isinstance(node, Ident):
if node.name not in self.scope: self.errors.append(
f"[SEMANTIK] Baris {node.line}: variabel '{node.name}' belum " f"dideklarasikan."
)
return "int"
return self.scope[node.name]
if isinstance(node, BinExpr):
lt = self._type_of(node.left) rt = self._type_of(node.right)
return "float" if "float" in (lt, rt) else "int"
raise SemanticError(f"Node ekspresi tidak dikenal: {node}")


# ===================================================================== # TAHAP 4: GENERASI KODE ANTARA (THREE-ADDRESS CODE)
# =====================================================================
# Skema penerjemahan deklarasi fungsi yang dipakai: #
#	FUNC_BEGIN nama_fungsi
#	; parameter: tipe nama (dianggap sudah terikat saat pemanggilan) #	<kode body>
#	RETURN <tempat_nilai> #	FUNC_END nama_fungsi
# =====================================================================

class CodeGenerator:
def   init  (self): self.instructions: List[str] = [] self._temp_count = 0

def _new_temp(self) -> str: self._temp_count += 1
return f"t{self._temp_count}"

def _emit(self, text: str) -> None: self.instructions.append(text)

def generate(self, func: FuncDecl) -> List[str]: self._emit(f"FUNC_BEGIN {func.name}")
for p in func.params:
self._emit(f"; param {p.ptype} {p.name}") for stmt in func.body:
self._gen_stmt(stmt) self._emit(f"FUNC_END {func.name}") return self.instructions

def _gen_stmt(self, node) -> None:
if isinstance(node, VarDecl):
self._emit(f"; local {node.vtype} {node.name}") elif isinstance(node, Assign):
place = self._gen_expr(node.expr) self._emit(f"{node.name} = {place}")
elif isinstance(node, Return):

place = self._gen_expr(node.expr) self._emit(f"RETURN {place}")

def _gen_expr(self, node) -> str:
if isinstance(node, Literal):
return str(node.value)
if isinstance(node, Ident):
return node.name
if isinstance(node, BinExpr):
left = self._gen_expr(node.left) right = self._gen_expr(node.right) temp = self._new_temp()
self._emit(f"{temp} = {left} {node.op} {right}") return temp
raise SemanticError(f"Node ekspresi tidak dikenal saat codegen: {node}")


# =====================================================================
# DRIVER: menjalankan seluruh pipeline kompilasi end-to-end
# =====================================================================

def compile_and_report(source: str, label: str = "PROGRAM") -> None: print("=" * 70)
print(f" SOURCE CODE - {label}") print("=" * 70) print(source.strip())

print("\n" + "=" * 70)
print(" TAHAP 1: ANALISIS LEKSIKAL")
print("=" * 70)
tokens = Lexer(source).tokenize() for t in tokens:
if t.kind != TokenKind.EOF: print(f"<{t.kind.name}:{t.lexeme!r}>")

print("\n" + "=" * 70)
print(" TAHAP 2: ANALISIS SINTAKSIS (AST)")
print("=" * 70)
ast = Parser(tokens).parse() print_tree(ast)

print("\n" + "=" * 70)
print(" TAHAP 3: ANALISIS SEMANTIK")
print("=" * 70)
analyzer = SemanticAnalyzer() try:
scope = analyzer.analyze(ast)
print("Status: LULUS. Tidak ditemukan kesalahan semantik.") print("Scope Fungsi (parameter + variabel lokal):")
for name, vtype in scope.items(): print(f" {name} : {vtype}")
except SemanticError as e:
print("Status: GAGAL. Ditemukan kesalahan semantik:") print(e)
return

print("\n" + "=" * 70)
print(" TAHAP 4: GENERASI KODE ANTARA (THREE-ADDRESS CODE)")
print("=" * 70)
tac = CodeGenerator().generate(ast) for line in tac:
print(line) print()


if   name   == "  main  ":
#
# CONTOH 1: Program valid - fungsi menghitung luas persegi panjang #
program_ok = """
int hitungLuas(int panjang, int lebar) { int luas;
luas = panjang * lebar; return luas;


