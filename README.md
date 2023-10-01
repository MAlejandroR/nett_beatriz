# Modificación proyecto origina de Beatriz

> en este Readme, indicaré las modificaciones concretas para conseguir que el cliente implementado con anbular autentifique contra una api de servidor implementado de laravel.
> 
> 
# Descargo el proyecto y actualizo con composer y creo la clave de la aplicación
````bash
git clone ....
cd /a la carpeta del back
//Creo el fichero .env
composer update
php artisan key:gen
````

> ## En el back
### Base de datos
> Primero configuro la base de datos. Para ello me he creado un docker-compose. Este paso no es para que funcione en mi máquina, en la tuya tendrá que conectar con una base de datos que tengas en tu máquina o en el plesk de nett  
> La configuración se realiza en el fichero __.env__
> 
#### Ejecutar migraciones

1.-  Hay que modificar el fichero ![./back/laravel-farmacias/database/migraciones/2023_09_24_085224_create_navbars_table.php]() eliminando una línea duplicada
```php
            $table->timestamps();
```
2.- Añado un fichero para que cree usuarios y así poder probar el login, luego ya agregaremos con el register   
Añadimos directamente la carpeta __seeder__ con los ficheros que hay ahí, básicamente son ficheros que se van a ejectuar cuando creemos la base de datos y así se crearán los ususarios   
Todos con el password __12345678__

3.- Ejecuto las migraciones y la población en la base de datos. El fresh es para que si hubiera tablas anteriores con esos nombres, que las borre  

````bash
php artisan migrate:fresh --seed
````
#### Creamos un middleware para aplicar el cors ()

* __CORS (Cross-Origin Resource Sharing)__ es un mecanismo que permite o restringe las solicitudes HTTP entre diferentes dominios en una API web.    
El nombre del middleware no es importante, en este caso le he llamado __cors__

```bash
php artisan make:middleware Cors
```
Esto nos habrá crado un  fichero [./app/Http/Middleware/cors.php](./app/Http/Middleware/cors.php)
Establecemos en el middleware a qué urls y con qué verbos damos permisos, en este caso vamos a dar permisos de forma generosa a cualquiera   
Para ello vamos al fichero y escribiemos en el método __hadler__ ya creado
```bash
 public function handle(Request $request, Closure $next): Response
    {
        return $next($request)
            ->header("Access-Control-Allow-Origin", "*")
            ->header("Access-Control-Allow-Methods", "PUT, POST, GET, DELETE")
            ->header("Access-Control-Allow-Headers", "Accept, Authorization, Content-Type");    }
}
```

Ahora tenemos que especificar que aplique el middleware a todas las rutas, o por lo menos a las rutas del api.    
Vamos a hacer de manera un poco salomónica y se lo aplicamos a todas   
En el fichero ![./app/Http/Kernel.php](), lo agretamos al array  de __middleware__

```php
    protected $middleware = [
        \App\Http\Middleware\Cors::class,
      // ... RESTO ....

```
#### Añadimos la rutas y el controlador

* Ahora tenemos que añaidir las rutas de login y register y el controlador correspondiente  
* En el fichero de rutas __![./routes/web.php]()__ (lo podíamos hacer también en el de api ...)
* Ponemos las rutas al final, ya que Auth::route(), agrega también las rutas de login y register, siendo que por lo visto, has instalado un sistema de autentificación en el servidor. para asegurar que no va a haber ningún conflicto, mejor comentarlo.
```php


//Auth::routes();
// ... RESTO ...
Route::post('/login',[\App\Http\Controllers\UserController::class,'login']);
Route::post('/resgister',[\App\Http\Controllers\UserController::class,'register']);
```

*Ahora creamos el controlador __UserController__ 
````php

php artisan make:controller UserController
````

*Vamos a la clase y escribimos el método __login__, no olvidar de incluir con use, las clases que vamos a utilizar. Este es el fallo que tuve el día de la tutoría
```php

use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;

class UserController extends Controller
{
    //

     public function login(Request $request)
    {
        $email = $request->email;
        $user = User::where('email', $email)->first();
        if (($user) &&((Hash::check($request->password, $user->password)))) {
                session(['email_'.$user->id=$email]);
                return response()->json(['Status' => 'Session Started']);
            } else {
                return response()->json(['Status' => 'User/Password No válido']);
            }
    }
```
> El código creo que se entiende fácilmente, si tieens dudas me comentas

#### última cosa
Larvel exige que si vienen datos de un formulario, haya un toke csrf, pero en este caso se debe de quedar excluída esta acción.    
Para ello, hay que especificarlo en ![./app/Http/Middleware/VerifyCsrfTokem.php]()

````php
  protected $except = [
        //
        "/login"
    ];

````



## El front

Lo primero vamos a la vista de tu componente de login: ![./src/app/auth/pages/login/login.component.html]()   
Ahí falta asociar el evento submit a un método de la parte del controlador que tampoco está. La idea es que cuando hagas un click en tu formulario se invoque a la apli del servidor. Vamos a ello

Modificamos la línea de la cabecera, asociando ahí el evento y poniendo un nombre al formulario para poder acceder a las cajas de texto en el controlador, así como poner nombre a las cajas de texto
````html
    <form autocomplete="off" [formGroup]="miFormulario" (ngSubmit)="login()">
    <!--RESTO DE FICHERO HTML ....-->
    <input matInput formControlName="email" type="email" placeholder="Email Usuario">
    </mat-form-field>
    <mat-form-field class="example-full-width">
        <input  formControlName="password" matInput type="password" placeholder="Contraseña">
        <!--RESTO DE FICHERO HTML ....-->

````
*En el fichero del controlador ![./src/app/auth/pages/login/login.component.ts]()
```javascript
import {Component, OnInit} from '@angular/core';
import { NgModule } from '@angular/core';
import {FormBuilder, FormGroup, Validators} from '@angular/forms';
import {Router} from '@angular/router';
import {AuthService} from '../../services/auth.service';



@NgModule({
  // ...
})
export class AppModule { }
@Component({
  selector: 'app-login',
  templateUrl: './login.component.html',
  styleUrls: ['./login.component.css']
})
export class LoginComponent implements  OnInit {
  constructor(private fb: FormBuilder,
              private AuthService: AuthService,
              private router: Router) {}

  miFormulario: FormGroup = this.fb.group({
    email: ['', [Validators.required]],
    password: ['', [Validators.required, Validators.minLength(6)]],
  });


  ngOnInit(){
  }


  login() {
    console.log(this.miFormulario.value);
    const {email, password} = this.miFormulario.value;

    console.log("Email :" +email+" Password "+password);
    this.AuthService.login(email,password).subscribe(data=>{
      console.log(data);
    });
    return false;
  }
}

```
* Ahora hay que agregar el método login en el servicio para poderse suscribir  a él (bueno esto habría que hacerlo antes, pero es igual...)
* También hay que cargar la variable loginUrl que se delcarará en el fichero de enviroment
Vamos al fichero ![src/app/auth/services/auth.service.ts]()
````javascript

export class AuthService {

    private baseUrl: string = environment.baseUrl;
    private loginUrl: string = environment.loginUrl;


    // Resto de código



  login (email: string, password:string){
    return  this.http.post(this.loginUrl,{email:email, password:password})
  }
````

Añadimos el valor de la variable de enviroment en el fichero ![./enviroments/enviroment.ts]()  y también lo dejamos preparado para el development ![./enviroments/enviroment.development.ts]()
También modificamos la variable baseURL, y quitamos lo de __api__ y dejamos todas las rutas en __web.php__ del servidor
````bash
//Lo que había y agregamos
baseUrl:'http://127.0.0.1:8000/register',
loginUrl:'http://127.0.0.1:8000/login'
````

Esto claro, cuando lo tengas desplegado tendrás que poner la url correspondiente. yo creo que tal cual lo tienes subido sería https://beatriz.proyectosdwa.es/app-login_v1/public/login


Ahora ya lo tienes, con el código que hay puedes ver en la consola lo que retorna el servidor. 
Lo que queda ahora es que uses el router de angular para que si el servidor me retorna el status ok pues redifijas al dashboard y si no te quedas ahí...
