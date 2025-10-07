// Proyecto: Plataforma de Empleo - Universidad de Cundinamarca (Sede Fusagasugá)
// Tipo: Aplicación modular (React frontend) + PHP (XAMPP) + MySQL
// Este archivo contiene: README, estructura de carpetas, esquema SQL, endpoints PHP esenciales, y componentes React iniciales con Tailwind.

/* ====================== README (Resumen rápido) ======================

Objetivo
--------
Crear una plataforma de empleo dirigida a estudiantes activos y egresados, y empresas. Funcionalidades mínimas:
- Registro y login de usuarios (Estudiantes y Empresas).
- Perfiles públicos con foto de perfil y foto de portada.
- Panel de empresa para publicar ofertas.
- Buscador y filtros de vacantes.
- Arquitectura modular con React (Tailwind CSS) + PHP (API REST simple) + MySQL (XAMPP).

Requisitos locales
------------------
- Node.js y npm
- XAMPP (Apache + MySQL + PHP)
- Composer (opcional)

Pasos rápidos
------------
1. Clona este repo o copia la estructura.
2. En XAMPP: coloca la carpeta `backend` dentro de `htdocs`.
3. Importa `database/schema.sql` en phpMyAdmin.
4. Inicia Apache y MySQL.
5. En `frontend`: `npm install` then `npm run dev` (o `npm start`).

======================================================================*/

/* ====================== Estructura sugerida ======================

/PlataformaEmpleo/
  /frontend/               <- React app (code/react)
    /src/
      /components/
        Navbar.jsx
        Auth/Login.jsx
        Auth/RegisterStudent.jsx
        Auth/RegisterCompany.jsx
        Profile/ProfileView.jsx
        Profile/ProfileEdit.jsx
        Jobs/JobList.jsx
        Jobs/JobCard.jsx
        Company/DashboardCompany.jsx
      /services/
        api.js            <- funciones fetch hacia backend
      App.jsx
      index.css           <- Tailwind import
  /backend/                <- PHP + endpoints
    /uploads/              <- fotos de perfil y portadas
    api.php               <- router simple
    auth.php
    register.php
    login.php
    profile.php
    jobs.php
    db.php                <- conexión a MySQL
  /database/
    schema.sql
  README.md

==================================================================*/

/* ====================== database/schema.sql ======================

-- Base de datos: plataforma_empleo
CREATE DATABASE IF NOT EXISTS plataforma_empleo CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE plataforma_empleo;

-- Tabla roles
CREATE TABLE IF NOT EXISTS roles (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(50) NOT NULL UNIQUE
);
INSERT IGNORE INTO roles (id, name) VALUES (1,'student'), (2,'company'), (3,'admin');

-- Usuarios
CREATE TABLE IF NOT EXISTS users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  role_id INT NOT NULL,
  email VARCHAR(255) NOT NULL UNIQUE,
  password VARCHAR(255) NOT NULL,
  full_name VARCHAR(255),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (role_id) REFERENCES roles(id)
);

-- Perfiles de estudiante
CREATE TABLE IF NOT EXISTS student_profiles (
  user_id INT PRIMARY KEY,
  phone VARCHAR(50),
  bio TEXT,
  degree VARCHAR(255),
  group_year INT,
  profile_image VARCHAR(255),
  cover_image VARCHAR(255),
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

-- Perfiles de empresa
CREATE TABLE IF NOT EXISTS company_profiles (
  user_id INT PRIMARY KEY,
  company_name VARCHAR(255),
  website VARCHAR(255),
  description TEXT,
  logo VARCHAR(255),
  cover_image VARCHAR(255),
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

-- Ofertas de trabajo
CREATE TABLE IF NOT EXISTS jobs (
  id INT AUTO_INCREMENT PRIMARY KEY,
  company_id INT NOT NULL,
  title VARCHAR(255) NOT NULL,
  description TEXT,
  location VARCHAR(255),
  salary VARCHAR(255),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (company_id) REFERENCES users(id) ON DELETE CASCADE
);

-- Postulaciones
CREATE TABLE IF NOT EXISTS applications (
  id INT AUTO_INCREMENT PRIMARY KEY,
  job_id INT NOT NULL,
  student_id INT NOT NULL,
  cover_letter TEXT,
  status ENUM('applied','viewed','rejected','accepted') DEFAULT 'applied',
  applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (job_id) REFERENCES jobs(id) ON DELETE CASCADE,
  FOREIGN KEY (student_id) REFERENCES users(id) ON DELETE CASCADE
);

/* ====================== Fin schema.sql ======================*/


/* ====================== backend/db.php ======================
<?php
$host = '127.0.0.1';
$db   = 'plataforma_empleo';
$user = 'root';
$pass = '';
$charset = 'utf8mb4';

$dsn = "mysql:host=$host;dbname=$db;charset=$charset";
$options = [
    PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,
    PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
    PDO::ATTR_EMULATE_PREPARES   => false,
];
try {
     $pdo = new PDO($dsn, $user, $pass, $options);
} catch (
PDOException $e) {
     http_response_code(500);
     echo json_encode(['error' => 'DB connection failed']);
     exit;
}
?>
*/

/* ====================== backend/register.php (simplificado) ======================
<?php
header('Content-Type: application/json');
require 'db.php';

$data = json_decode(file_get_contents('php://input'), true);
if(!$data) { http_response_code(400); echo json_encode(['error'=>'Invalid JSON']); exit; }

$email = $data['email'] ?? null;
$password = $data['password'] ?? null;
$role = $data['role'] ?? 'student';

if(!$email || !$password) { http_response_code(400); echo json_encode(['error'=>'Missing fields']); exit; }

// validar email...
$hashed = password_hash($password, PASSWORD_DEFAULT);
// obtener role_id
$stmt = $pdo->prepare('SELECT id FROM roles WHERE name = ? LIMIT 1');
$stmt->execute([$role]);
$r = $stmt->fetch();
$role_id = $r ? $r['id'] : 1;

try {
  $stmt = $pdo->prepare('INSERT INTO users (role_id,email,password,full_name) VALUES (?,?,?,?)');
  $stmt->execute([$role_id, $email, $hashed, $data['full_name'] ?? null]);
  $user_id = $pdo->lastInsertId();
  echo json_encode(['ok'=>true,'user_id'=>$user_id]);
} catch (PDOException $e) {
  http_response_code(500);
  echo json_encode(['error'=>'DB error']);
}
?>
*/

/* ====================== backend/login.php (simplificado) ======================
<?php
header('Content-Type: application/json');
require 'db.php';
$data = json_decode(file_get_contents('php://input'), true);
$email = $data['email'] ?? null;
$password = $data['password'] ?? null;
if(!$email || !$password){ http_response_code(400); echo json_encode(['error'=>'Missing']); exit; }
$stmt = $pdo->prepare('SELECT id,password,role_id FROM users WHERE email = ? LIMIT 1');
$stmt->execute([$email]);
$user = $stmt->fetch();
if(!$user || !password_verify($password, $user['password'])){ http_response_code(401); echo json_encode(['error'=>'Invalid credentials']); exit; }
// Crear sesión simple o JWT (recomendado JWT para SPA)
// Aquí devolvemos user id y role
echo json_encode(['ok'=>true,'user_id'=>$user['id'],'role_id'=>$user['role_id']]);
?>
*/

/* ====================== backend/profile.php (GET and POST profile) ======================
<?php
header('Content-Type: application/json');
require 'db.php';
$method = $_SERVER['REQUEST_METHOD'];
if($method === 'GET'){
  $id = $_GET['id'] ?? null;
  if(!$id){ http_response_code(400); echo json_encode(['error'=>'Missing id']); exit; }
  // mirar role
  $stmt = $pdo->prepare('SELECT role_id FROM users WHERE id=?'); $stmt->execute([$id]); $u=$stmt->fetch();
  if(!$u) { http_response_code(404); echo json_encode(['error'=>'User not found']); exit; }
  if($u['role_id']==1){
    $stmt = $pdo->prepare('SELECT u.id,u.email,u.full_name,s.* FROM users u JOIN student_profiles s ON u.id=s.user_id WHERE u.id=?');
    $stmt->execute([$id]); $res = $stmt->fetch(); echo json_encode($res);
  } else {
    $stmt = $pdo->prepare('SELECT u.id,u.email,u.full_name,c.* FROM users u JOIN company_profiles c ON u.id=c.user_id WHERE u.id=?');
    $stmt->execute([$id]); $res = $stmt->fetch(); echo json_encode($res);
  }
}
// POST handling omitted (file uploads) - implement using multipart/form-data and move_uploaded_file()
?>
*/

/* ====================== frontend/src/services/api.js ======================
// Servicio fetch básico
const API_BASE = 'http://localhost/backend';

export async function register(payload){
  const res = await fetch(`${API_BASE}/register.php`, {
    method:'POST', headers:{'Content-Type':'application/json'}, body: JSON.stringify(payload)
  });
  return res.json();
}

export async function login(payload){
  const res = await fetch(`${API_BASE}/login.php`,{
    method:'POST', headers:{'Content-Type':'application/json'}, body: JSON.stringify(payload)
  });
  return res.json();
}

export async function getProfile(id){
  const res = await fetch(`${API_BASE}/profile.php?id=${id}`);
  return res.json();
}
*/

/* ====================== frontend/src/App.jsx (skeleton) ======================
import React from 'react';
import { BrowserRouter as Router, Routes, Route } from 'react-router-dom';
import Navbar from './components/Navbar';
import Login from './components/Auth/Login';
import RegisterStudent from './components/Auth/RegisterStudent';
import RegisterCompany from './components/Auth/RegisterCompany';
import ProfileView from './components/Profile/ProfileView';

export default function App(){
  return (
    <Router>
      <div className="min-h-screen bg-gray-50">
        <Navbar />
        <main className="container mx-auto p-4">
          <Routes>
            <Route path="/" element={<div>Inicio - buscador de empleos</div>} />
            <Route path="/login" element={<Login />} />
            <Route path="/register/student" element={<RegisterStudent />} />
            <Route path="/register/company" element={<RegisterCompany />} />
            <Route path="/profile/:id" element={<ProfileView />} />
          </Routes>
        </main>
      </div>
    </Router>
  )
}
*/

/* ====================== frontend/src/components/Auth/RegisterStudent.jsx ======================
import React, {useState} from 'react';
import { register } from '../../services/api';
import { useNavigate } from 'react-router-dom';

export default function RegisterStudent(){
  const [email,setEmail]=useState('');
  const [password,setPassword]=useState('');
  const [fullName,setFullName]=useState('');
  const navigate = useNavigate();

  const handleSubmit = async (e)=>{
    e.preventDefault();
    const resp = await register({email,password,full_name:fullName,role:'student'});
    if(resp.ok) navigate('/login'); else alert('Error: '+(resp.error||'')); 
  }

  return (
    <div className="max-w-md mx-auto bg-white p-6 rounded-lg shadow">
      <h2 className="text-xl font-semibold mb-4">Registro - Estudiante</h2>
      <form onSubmit={handleSubmit} className="space-y-3">
        <input value={fullName} onChange={e=>setFullName(e.target.value)} placeholder="Nombre completo" className="w-full p-2 border rounded" />
        <input value={email} onChange={e=>setEmail(e.target.value)} placeholder="Correo" className="w-full p-2 border rounded" />
        <input type="password" value={password} onChange={e=>setPassword(e.target.value)} placeholder="Contraseña" className="w-full p-2 border rounded" />
        <button className="w-full py-2 rounded bg-indigo-600 text-white">Registrarme</button>
      </form>
    </div>
  )
}
*/

/* ====================== frontend/src/components/Auth/Login.jsx ======================
import React, {useState} from 'react';
import { login } from '../../services/api';
import { useNavigate } from 'react-router-dom';

export default function Login(){
  const [email,setEmail]=useState('');
  const [password,setPassword]=useState('');
  const navigate = useNavigate();
  const handleSubmit = async (e)=>{
    e.preventDefault();
    const res = await login({email,password});
    if(res.ok){
      // guardar en localStorage (temporal) - usar JWT en producción
      localStorage.setItem('user_id', res.user_id);
      localStorage.setItem('role_id', res.role_id);
      navigate(`/profile/${res.user_id}`);
    } else alert('Credenciales invalidas');
  }
  return (
    <div className="max-w-md mx-auto bg-white p-6 rounded-lg shadow">
      <h2 className="text-xl font-semibold mb-4">Iniciar sesión</h2>
      <form onSubmit={handleSubmit} className="space-y-3">
        <input value={email} onChange={e=>setEmail(e.target.value)} placeholder="Correo" className="w-full p-2 border rounded" />
        <input type="password" value={password} onChange={e=>setPassword(e.target.value)} placeholder="Contraseña" className="w-full p-2 border rounded" />
        <button className="w-full py-2 rounded bg-green-600 text-white">Entrar</button>
      </form>
    </div>
  )
}
*/

/* ====================== frontend/src/components/Profile/ProfileView.jsx ======================
import React, {useEffect, useState} from 'react';
import { useParams } from 'react-router-dom';
import { getProfile } from '../../services/api';

export default function ProfileView(){
  const { id } = useParams();
  const [profile, setProfile] = useState(null);
  useEffect(()=>{ (async ()=>{ const res = await getProfile(id); setProfile(res); })(); },[id]);

  if(!profile) return <div>Cargando...</div>;

  const cover = profile.cover_image ? `/backend/uploads/${profile.cover_image}` : null;
  const avatar = profile.profile_image || profile.logo ? `/backend/uploads/${profile.profile_image||profile.logo}` : null;

  return (
    <div className="bg-white rounded shadow">
      <div className="h-48 bg-gray-200" style={{backgroundImage: cover?`url(${cover})`:'none', backgroundSize:'cover', backgroundPosition:'center'}}></div>
      <div className="p-4 flex gap-4">
        <img src={avatar} alt="avatar" className="w-28 h-28 rounded-full -mt-16 border-4 border-white" />
        <div>
          <h2 className="text-2xl font-bold">{profile.full_name || profile.company_name}</h2>
          <p>{profile.bio || profile.description}</p>
        </div>
      </div>
    </div>
  )
}
*/

/* ====================== Notas de seguridad y siguientes pasos ======================
- Usar HTTPS (en producción) y JWT para autenticación en SPA.
- Validar y sanitizar todos los inputs en backend.
- Implementar subida segura de archivos (validar tipo y tamaño).
- Añadir paginación y filtros para ofertas.
- Crear role-based routing y paneles privados.
- Considerar usar Laravel (PHP) o Node.js + Express para mayor estructura.

==================================================================*/

// Fin del documento. Puedes usar este esqueleto para empezar y pedirme que te genere
// archivos concretos (por ejemplo: App.jsx completo, login.php, export SQL listo) y los creo.
