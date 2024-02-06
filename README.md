// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/utils/math/SafeMath.sol";

contract PrestamoDeFi {
    using SafeMath for uint256;

    struct Prestamo {
        uint id;
        uint monto;  // monto del préstamo en wei
        uint plazo;  // plazo del préstamo en segundos 
        uint tiempoSolicitud;   // timestamp de la solicitud
        uint tiempoLimite;  // timestamp límite para reembolso
        bool aprobado;
        bool reembolsado;
        bool liquidado;
        address prestatario;  // dirección del solicitante del préstamo
    }
        //stuctura de datos para comprobar
    struct Cliente {
        bool activado;
        uint saldoGarantia; // saldo de garantía en wei 
        mapping(uint => Prestamo) prestamos;
        uint[] prestamoIds;
    }

    address public socioPrincipal;
    mapping(address => Cliente) public clientes;
    mapping(address => bool) empleadosPrestamista;

    event SolicitudPrestamo(uint id, address indexed prestatario, uint monto, uint plazo);
    event PrestamoAprobado(uint id, address indexed prestatario, uint monto);
    event PrestamoReembolsado(uint id, address indexed prestatario, uint monto);
    event GarantiaLiquidada(uint id, address indexed prestatario);

    modifier soloSocioPrincipal() {
        require(msg.sender == socioPrincipal, "Solo el socio principal puede realizar esta accion");
        _;
    }
        // estrutuctura  para comprobar 
    modifier soloEmpleadoPrestamista() {
        require(empleadosPrestamista[msg.sender], "Solo un empleado prestamista puede realizar esta accion");
        _;
    }

    modifier soloClienteRegistrado() {
        require(clientes[msg.sender].activado, "Cliente no registrado");
        _;
    }

    constructor() {
        socioPrincipal = msg.sender;
        empleadosPrestamista[socioPrincipal] = true;
    }

    function altaPrestamista(address nuevoPrestamista) public soloSocioPrincipal {
        require(!empleadosPrestamista[nuevoPrestamista], "El prestamista ya esta dado de alta");
        empleadosPrestamista[nuevoPrestamista] = true;
    }

    function altaCliente(address nuevoCliente) public soloEmpleadoPrestamista {
        require(!clientes[nuevoCliente].activado, "El cliente ya esta dado de alta");
        clientes[nuevoCliente].activado = true;
        clientes[nuevoCliente].saldoGarantia = 0;
    }

    function depositarGarantia() public payable soloClienteRegistrado {
        require(msg.value > 0, "Debe enviar una cantidad positiva de Ether");
        clientes[msg.sender].saldoGarantia = clientes[msg.sender].saldoGarantia.add(msg.value); //clientes(msg.sender).saldoGarantia += msg.value;
    }

    function solicitarPrestamo(uint monto_, uint plazo_) public soloClienteRegistrado returns (uint) {
        Cliente storage cliente = clientes[msg.sender];
        require(cliente.saldoGarantia >= monto_, "Saldo de garantia insuficiente");

        uint nuevoId = cliente.prestamoIds.length + 1;
        cliente.prestamoIds.push(nuevoId);

        Prestamo storage nuevoPrestamo = cliente.prestamos[nuevoId];
        nuevoPrestamo.id = nuevoId;
        nuevoPrestamo.monto = monto_; // Asegúrar de enviar el monto en wei        
        nuevoPrestamo.tiempoLimite = 0; // Se actualizara a aprobar el prestamo
        nuevoPrestamo.plazo = plazo_;
        nuevoPrestamo.tiempoSolicitud = block.timestamp;
        nuevoPrestamo.aprobado = false;
        nuevoPrestamo.reembolsado = false;
        nuevoPrestamo.liquidado = false;
        nuevoPrestamo.prestatario = msg.sender;

        emit SolicitudPrestamo(nuevoId, msg.sender, monto_, plazo_);

        return nuevoId;
    }

    function aprobarPrestamo(uint id_, address prestatario_) public soloEmpleadoPrestamista {
        Prestamo storage prestamo = clientes[prestatario_].prestamos[id_];
        require(prestamo.id != 0 && !prestamo.aprobado && !prestamo.liquidado, "Prestamo no valido");
        require(prestamo.id != 0 && !prestamo.aprobado, "Prestamo no valido o ya aprobado");

        prestamo.aprobado = true;
        prestamo.tiempoLimite = block.timestamp.add(prestamo.plazo);

        emit PrestamoAprobado(id_, prestatario_, prestamo.monto);
    }

    function reembolsarPrestamo(uint id_) public soloClienteRegistrado {
        Cliente storage cliente = clientes[msg.sender];
        Prestamo storage prestamo = cliente.prestamos[id_];

        require(prestamo.prestatario == msg.sender, "Solo el prestatario puede reembolsar el prestamo");
        require(prestamo.aprobado, "El prestamo no esta aprobado");
        require(!prestamo.reembolsado, "El prestamo ya ha sido reembolsado");
        require(!prestamo.liquidado, "El prestamo ya ha sido liquidado");
        require(prestamo.tiempoLimite >= block.timestamp, "El tiempo para reembolsar el prestamo ha vencido");
        require(cliente.saldoGarantia >= prestamo.monto, "Saldo de garantia insuficiente");
        

        payable(socioPrincipal).transfer(prestamo.monto);
        cliente.saldoGarantia = cliente.saldoGarantia - prestamo.monto;
        prestamo.reembolsado = true;

        emit PrestamoReembolsado(id_, msg.sender, prestamo.monto);
    }
        //funcion de datos para comprobar 
    function obtenerPrestamosPorPrestatario(address prestatario_) public view returns (uint[] memory) {
        return clientes[prestatario_].prestamoIds;
    }
        //funcion de datos para comprobar 
    function obtenerDetalleDePrestamo(uint id_) public view returns (Prestamo memory) {  
        Cliente storage cliente = clientes[msg.sender];
        require(cliente.prestamos[id_].id != 0, "Prestamo no existe");
        require(cliente.prestamos[id_].prestatario == msg.sender || msg.sender == socioPrincipal || empleadosPrestamista[msg.sender], "No autorizado para ver este prestamo");
        
        return cliente.prestamos[id_];
    }
     //  función liquidarGarantia
    function liquidarGarantia(uint id_, address prestatario) public soloEmpleadoPrestamista {
        Prestamo storage prestamo = clientes[prestatario].prestamos[id_];
        
        require(prestamo.id != 0, "El prestamo no existe");
        require(!prestamo.reembolsado, "El prestamo ya ha sido reembolsado");
        require(block.timestamp > prestamo.tiempoLimite, "El prestamo no ha vencido aun");
        require(!prestamo.liquidado, "El prestamo ya ha sido liquidado");
       
        uint montoGarantia = clientes[prestatario].saldoGarantia;
        payable(socioPrincipal).transfer(montoGarantia);
        
        prestamo.liquidado = true;

      // Actualizar el saldo de garantía del prestatario, dejando el remanente
        clientes[prestatario].saldoGarantia  -= montoGarantia ;
        
        emit GarantiaLiquidada(id_, prestatario);
    }  
}


