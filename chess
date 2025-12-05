import pygame
import sys
import os
import socket
import threading
import json
import time

# Initialize pygame
pygame.init()
pygame.mixer.init()

# Screen dimensions
WIDTH, HEIGHT = 640, 640
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("Chess Game")

# Colors
DARK_SQUARE = (118, 150, 86)
LIGHT_SQUARE = (238, 238, 210)
HIGHLIGHT = (186, 202, 68)
MOVE_HIGHLIGHT = (124, 174, 221, 180)
SELECTED_SQUARE = (124, 174, 221)
CHECK_HIGHLIGHT = (220, 20, 60, 180)
TEXT_COLOR = (50, 50, 50)
WHITE = (255, 255, 255)
BLACK = (0, 0, 0)
BUTTON_COLOR = (70, 130, 180)
BUTTON_HOVER = (100, 149, 237)
INPUT_BG = (240, 240, 240)
INPUT_BORDER = (200, 200, 200)

# Game constants
SQUARE_SIZE = WIDTH // 8
PIECE_SIZE = SQUARE_SIZE - 10

# Font
font = pygame.font.SysFont(None, 36)
small_font = pygame.font.SysFont(None, 24)
title_font = pygame.font.SysFont(None, 48)

def resource_path(relative_path):
    try:
        base_path = sys._MEIPASS
    except Exception:
        base_path = os.path.abspath(".")
    return os.path.join(base_path, relative_path)

# Load sound (if available)
move_sound = None

class Piece:
    def __init__(self, color, piece_type, row, col):
        self.color = color  # 'white' or 'black'
        self.piece_type = piece_type  # 'pawn', 'rook', 'knight', 'bishop', 'queen', 'king'
        self.row = row
        self.col = col
        self.has_moved = False
    
    def get_symbol(self):
        symbols = {
            'king': 'K',
            'queen': 'Q',
            'rook': 'R',
            'bishop': 'B',
            'knight': 'N',
            'pawn': 'P'
        }
        return symbols[self.piece_type]
    
    def draw(self, screen, rotated=False):
        # Create text surface for the piece
        color = (255, 255, 255) if self.color == 'white' else (0, 0, 0)
        symbol = self.get_symbol()
        
        # Calculate position based on rotation
        if rotated:
            center_x = (7 - self.col) * SQUARE_SIZE + SQUARE_SIZE // 2
            center_y = (7 - self.row) * SQUARE_SIZE + SQUARE_SIZE // 2
        else:
            center_x = self.col * SQUARE_SIZE + SQUARE_SIZE // 2
            center_y = self.row * SQUARE_SIZE + SQUARE_SIZE // 2
        
        # Draw piece background circle
        pygame.draw.circle(screen, (200, 200, 200) if self.color == 'white' else (50, 50, 50), 
                          (center_x, center_y), PIECE_SIZE // 2)
        pygame.draw.circle(screen, (0, 0, 0) if self.color == 'white' else (255, 255, 255), 
                          (center_x, center_y), PIECE_SIZE // 2, 2)
        
        # Draw piece symbol
        text = font.render(symbol, True, color)
        text_rect = text.get_rect(center=(center_x, center_y))
        screen.blit(text, text_rect)

class Button:
    def __init__(self, x, y, width, height, text, color=BUTTON_COLOR):
        self.rect = pygame.Rect(x, y, width, height)
        self.text = text
        self.color = color
        self.hover_color = BUTTON_HOVER
        self.is_hovered = False
    
    def draw(self, screen):
        color = self.hover_color if self.is_hovered else self.color
        pygame.draw.rect(screen, color, self.rect, border_radius=10)
        pygame.draw.rect(screen, WHITE, self.rect, 2, border_radius=10)
        
        text_surf = font.render(self.text, True, WHITE)
        text_rect = text_surf.get_rect(center=self.rect.center)
        screen.blit(text_surf, text_rect)
    
    def check_hover(self, pos):
        self.is_hovered = self.rect.collidepoint(pos)
        return self.is_hovered
    
    def is_clicked(self, pos, event):
        if event.type == pygame.MOUSEBUTTONDOWN and event.button == 1:
            return self.rect.collidepoint(pos)
        return False

class InputBox:
    def __init__(self, x, y, w, h, text=''):
        self.rect = pygame.Rect(x, y, w, h)
        self.color = INPUT_BORDER
        self.text = text
        self.txt_surface = font.render(text, True, BLACK)
        self.active = False

    def handle_event(self, event):
        if event.type == pygame.MOUSEBUTTONDOWN:
            # If the user clicked on the input_box rect
            if self.rect.collidepoint(event.pos):
                # Toggle the active variable
                self.active = not self.active
            else:
                self.active = False
            # Change the current color of the input box
            self.color = INPUT_BORDER if not self.active else BUTTON_HOVER
        if event.type == pygame.KEYDOWN:
            if self.active:
                if event.key == pygame.K_RETURN:
                    self.active = False
                    self.color = INPUT_BORDER
                elif event.key == pygame.K_BACKSPACE:
                    self.text = self.text[:-1]
                else:
                    self.text += event.unicode
                # Re-render the text
                self.txt_surface = font.render(self.text, True, BLACK)

    def update(self):
        # Resize the box if the text is too long
        width = max(200, self.txt_surface.get_width()+10)
        self.rect.w = width

    def draw(self, screen):
        # Draw the input box
        pygame.draw.rect(screen, INPUT_BG, self.rect, border_radius=5)
        pygame.draw.rect(screen, self.color, self.rect, 2, border_radius=5)
        screen.blit(self.txt_surface, (self.rect.x+5, self.rect.y+5))

class ChessGame:
    def __init__(self):
        self.board = [[None for _ in range(8)] for _ in range(8)]
        self.current_player = 'white'
        self.selected_piece = None
        self.valid_moves = []
        self.game_over = False
        self.winner = None
        self.check = False
        self.checkmate = False
        self.stalemate = False
        self.move_history = []
        self.en_passant_target = None  # Position where en passant is possible
        self.player_side = None  # 'white', 'black', or None for local play
        self.game_mode = 'menu'  # 'menu', 'setup', 'playing', 'game_over'
        self.network_mode = None  # 'server', 'client', or None
        self.server_socket = None
        self.client_socket = None
        self.connection = None
        self.is_my_turn = False
        self.view_rotated = False  # For black player view
        self.move_count = 0  # Track total moves
        self.initialize_board()
    
    def initialize_board(self):
        # Set up pawns
        for col in range(8):
            self.board[1][col] = Piece('black', 'pawn', 1, col)
            self.board[6][col] = Piece('white', 'pawn', 6, col)
        
        # Set up other pieces
        # Black pieces (top)
        self.board[0][0] = Piece('black', 'rook', 0, 0)
        self.board[0][1] = Piece('black', 'knight', 0, 1)
        self.board[0][2] = Piece('black', 'bishop', 0, 2)
        self.board[0][3] = Piece('black', 'queen', 0, 3)
        self.board[0][4] = Piece('black', 'king', 0, 4)
        self.board[0][5] = Piece('black', 'bishop', 0, 5)
        self.board[0][6] = Piece('black', 'knight', 0, 6)
        self.board[0][7] = Piece('black', 'rook', 0, 7)
        
        # White pieces (bottom)
        self.board[7][0] = Piece('white', 'rook', 7, 0)
        self.board[7][1] = Piece('white', 'knight', 7, 1)
        self.board[7][2] = Piece('white', 'bishop', 7, 2)
        self.board[7][3] = Piece('white', 'queen', 7, 3)
        self.board[7][4] = Piece('white', 'king', 7, 4)
        self.board[7][5] = Piece('white', 'bishop', 7, 5)
        self.board[7][6] = Piece('white', 'knight', 7, 6)
        self.board[7][7] = Piece('white', 'rook', 7, 7)
    
    def draw_board(self, screen):
        # Draw squares
        for row in range(8):
            for col in range(8):
                # Calculate position based on rotation
                if self.view_rotated:
                    draw_row = 7 - row
                    draw_col = 7 - col
                else:
                    draw_row = row
                    draw_col = col
                
                color = LIGHT_SQUARE if (row + col) % 2 == 0 else DARK_SQUARE
                pygame.draw.rect(screen, color, (draw_col * SQUARE_SIZE, draw_row * SQUARE_SIZE, SQUARE_SIZE, SQUARE_SIZE))
        
        # Highlight selected square
        if self.selected_piece:
            # Calculate position based on rotation
            if self.view_rotated:
                draw_row = 7 - self.selected_piece.row
                draw_col = 7 - self.selected_piece.col
            else:
                draw_row = self.selected_piece.row
                draw_col = self.selected_piece.col
            
            pygame.draw.rect(screen, SELECTED_SQUARE, 
                            (draw_col * SQUARE_SIZE, 
                             draw_row * SQUARE_SIZE, 
                             SQUARE_SIZE, SQUARE_SIZE))
        
        # Highlight valid moves
        for move in self.valid_moves:
            row, col = move
            # Calculate position based on rotation
            if self.view_rotated:
                draw_row = 7 - row
                draw_col = 7 - col
            else:
                draw_row = row
                draw_col = col
            
            s = pygame.Surface((SQUARE_SIZE, SQUARE_SIZE), pygame.SRCALPHA)
            s.fill(MOVE_HIGHLIGHT)
            screen.blit(s, (draw_col * SQUARE_SIZE, draw_row * SQUARE_SIZE))
        
        # Highlight king in check
        if self.check:
            king_pos = self.find_king(self.current_player)
            if king_pos:
                row, col = king_pos
                # Calculate position based on rotation
                if self.view_rotated:
                    draw_row = 7 - row
                    draw_col = 7 - col
                else:
                    draw_row = row
                    draw_col = col
                
                s = pygame.Surface((SQUARE_SIZE, SQUARE_SIZE), pygame.SRCALPHA)
                s.fill(CHECK_HIGHLIGHT)
                screen.blit(s, (draw_col * SQUARE_SIZE, draw_row * SQUARE_SIZE))
        
        # Draw pieces
        for row in range(8):
            for col in range(8):
                piece = self.board[row][col]
                if piece:
                    piece.draw(screen, self.view_rotated)
        
        # Draw coordinates
        for i in range(8):
            # Numbers and letters based on rotation
            if self.view_rotated:
                # Numbers on the right
                text = small_font.render(str(i+1), True, TEXT_COLOR)
                screen.blit(text, (WIDTH - 20, (7 - i) * SQUARE_SIZE + 5))
                # Letters on the top
                text = small_font.render(chr(104-i), True, TEXT_COLOR)
                screen.blit(text, ((7 - i) * SQUARE_SIZE + 5, 5))
            else:
                # Numbers on the left
                text = small_font.render(str(8-i), True, TEXT_COLOR)
                screen.blit(text, (5, i * SQUARE_SIZE + 5))
                # Letters on the bottom
                text = small_font.render(chr(97+i), True, TEXT_COLOR)
                screen.blit(text, (i * SQUARE_SIZE + SQUARE_SIZE - 15, HEIGHT - 20))
    
    def get_board_position(self, mouse_x, mouse_y):
        """Convert mouse coordinates to board position, accounting for rotation"""
        col = mouse_x // SQUARE_SIZE
        row = mouse_y // SQUARE_SIZE
        
        # Adjust for rotation if needed
        if self.view_rotated:
            row = 7 - row
            col = 7 - col
            
        return row, col
    
    def select_piece(self, row, col):
        piece = self.board[row][col]
        if piece and piece.color == self.current_player:
            self.selected_piece = piece
            self.valid_moves = self.get_valid_moves(piece)
            return True
        return False
    def side_opponet_pawn(self,piece_):
        # print(piece_.col,piece_.row,self.board[piece_.col-1][piece_.row])
        if piece_.col>0:
            if(self.board[piece_.row][piece_.col-1]) != None:
                print("passant")
                return True
        if piece_.col<7:
            if (self.board[piece_.row][piece_.col+1]) != None and self.board[piece_.row][piece_.col+1].color!= piece_.color:
                print("passant")
                return True
        print("non passant")
        return False
    def move_piece(self, to_row, to_col):
        
        if not self.selected_piece:
            return False
        
        if (to_row, to_col) not in self.valid_moves:
            return False
        
        # Save state for undo and en passant tracking
        from_row, from_col = self.selected_piece.row, self.selected_piece.col
        captured_piece = self.board[to_row][to_col]
        
        # Handle en passant capture
        en_passant_capture = False
        if (self.selected_piece.piece_type == 'pawn' and 
            self.en_passant_target and 
            to_row == self.en_passant_target[0] and 
            to_col == self.en_passant_target[1]):
            # Capture the pawn that moved two squares
            captured_pawn_row = from_row  # Same row as the moving pawn was
            captured_pawn_col = to_col    # Same column as the target
            captured_piece = self.board[captured_pawn_row][captured_pawn_col]
            self.board[captured_pawn_row][captured_pawn_col] = None
            en_passant_capture = True
        
        # Handle castling
        castling = False
        if self.selected_piece.piece_type == 'king' and abs(to_col - from_col) == 2:
            castling = True
            # Determine which rook to move
            if to_col > from_col:  # Kingside castling
                rook_from_col = 7
                rook_to_col = to_col - 1
            else:  # Queenside castling
                rook_from_col = 0
                rook_to_col = to_col + 1
            
            # Move the rook
            rook = self.board[from_row][rook_from_col]
            self.board[from_row][rook_from_col] = None
            self.board[from_row][rook_to_col] = rook
            rook.col = rook_to_col
            rook.has_moved = True
        
        # Move the piece
        self.board[from_row][from_col] = None
        self.board[to_row][to_col] = self.selected_piece
        self.selected_piece.row = to_row
        self.selected_piece.col = to_col
        self.selected_piece.has_moved = True
        
        # Handle pawn two-square move (set en passant target)
        
        if (self.selected_piece.piece_type == 'pawn' and 
            abs(to_row - from_row) == 2 ):
            # Set en passant target to the square behind the pawn
            
            self.en_passant_target = ((from_row + to_row) // 2, to_col)
        else:
            self.en_passant_target = None
        # Play sound
        if move_sound:
            move_sound.play()
        
        # Check for pawn promotion
        if self.selected_piece.piece_type == 'pawn':
            if (self.selected_piece.color == 'white' and to_row == 0) or \
               (self.selected_piece.color == 'black' and to_row == 7):
                self.promote_pawn(self.selected_piece)
        
        # Switch players
        self.current_player = 'black' if self.current_player == 'white' else 'white'
        self.move_count += 1
        if self.network_mode and self.player_side:
            self.is_my_turn = (self.player_side == self.current_player)
        # Check for check/checkmate
        self.check = self.is_in_check(self.current_player)
        if self.check:
            if self.is_checkmate(self.current_player):
                self.checkmate = True
                self.game_over = True
                self.winner = 'black' if self.current_player == 'white' else 'white'
            elif self.is_stalemate(self.current_player):
                self.stalemate = True
                self.game_over = True
        elif self.is_stalemate(self.current_player):
            self.stalemate = True
            self.game_over = True
        
        # Save move to history
        self.move_history.append({
            'from': (from_row, from_col),
            'to': (to_row, to_col),
            'piece': self.selected_piece,
            'captured': captured_piece,
            'en_passant': en_passant_capture,
            'castling': castling
        })
        
        # Reset selection
        self.selected_piece = None
        self.valid_moves = []
        return True
    
    def promote_pawn(self, pawn):
        # For simplicity, always promote to queen
        pawn.piece_type = 'queen'
    
    def get_valid_moves(self, piece):
        moves = []
        row, col = piece.row, piece.col
        
        if piece.piece_type == 'pawn':
            direction = -1 if piece.color == 'white' else 1
            
            # Move forward
            if 0 <= row + direction < 8 and not self.board[row + direction][col]:
                moves.append((row + direction, col))
                
                # Initial double move
                if not piece.has_moved:
                    if 0 <= row + 2 * direction < 8 and not self.board[row + 2 * direction][col]:
                        moves.append((row + 2 * direction, col))
            
            # Capture diagonally (not horizontally or vertically)
            for dc in [-1, 1]:
                if 0 <= row + direction < 8 and 0 <= col + dc < 8:
                    target = self.board[row + direction][col + dc]
                    # Normal capture
                    if target and target.color != piece.color:
                        moves.append((row + direction, col + dc))
                    # En passant capture
                    elif (self.en_passant_target != None and 
                          row + direction == self.en_passant_target[0] and 
                          col + dc == self.en_passant_target[1]):
                        moves.append((row + direction, col + dc))
        
        elif piece.piece_type == 'rook':
            # Horizontal and vertical moves
            for dr, dc in [(0, 1), (1, 0), (0, -1), (-1, 0)]:
                for i in range(1, 8):
                    r, c = row + i * dr, col + i * dc
                    if not (0 <= r < 8 and 0 <= c < 8):
                        break
                    target = self.board[r][c]
                    if not target:
                        moves.append((r, c))
                    elif target.color != piece.color:
                        moves.append((r, c))
                        break
                    else:
                        break
        
        elif piece.piece_type == 'knight':
            # L-shaped moves
            for dr, dc in [(2, 1), (1, 2), (-1, 2), (-2, 1), (-2, -1), (-1, -2), (1, -2), (2, -1)]:
                r, c = row + dr, col + dc
                if 0 <= r < 8 and 0 <= c < 8:
                    target = self.board[r][c]
                    if not target or target.color != piece.color:
                        moves.append((r, c))
        
        elif piece.piece_type == 'bishop':
            # Diagonal moves
            for dr, dc in [(1, 1), (1, -1), (-1, 1), (-1, -1)]:
                for i in range(1, 8):
                    r, c = row + i * dr, col + i * dc
                    if not (0 <= r < 8 and 0 <= c < 8):
                        break
                    target = self.board[r][c]
                    if not target:
                        moves.append((r, c))
                    elif target.color != piece.color:
                        moves.append((r, c))
                        break
                    else:
                        break
        
        elif piece.piece_type == 'queen':
            # Combination of rook and bishop moves
            for dr, dc in [(0, 1), (1, 0), (0, -1), (-1, 0), (1, 1), (1, -1), (-1, 1), (-1, -1)]:
                for i in range(1, 8):
                    r, c = row + i * dr, col + i * dc
                    if not (0 <= r < 8 and 0 <= c < 8):
                        break
                    target = self.board[r][c]
                    if not target:
                        moves.append((r, c))
                    elif target.color != piece.color:
                        moves.append((r, c))
                        break
                    else:
                        break
        
        elif piece.piece_type == 'king':
            # One square in any direction
            for dr in [-1, 0, 1]:
                for dc in [-1, 0, 1]:
                    if dr == 0 and dc == 0:
                        continue
                    r, c = row + dr, col + dc
                    if 0 <= r < 8 and 0 <= c < 8:
                        target = self.board[r][c]
                        if not target or target.color != piece.color:
                            moves.append((r, c))
            
            # Castling
            if not piece.has_moved and not self.is_in_check(piece.color):
                # Kingside castling
                if (self.board[row][7] and 
                    self.board[row][7].piece_type == 'rook' and 
                    not self.board[row][7].has_moved and
                    not self.board[row][5] and 
                    not self.board[row][6]):
                    # Check if squares are not under attack
                    if not self.is_square_attacked(row, 5, piece.color) and not self.is_square_attacked(row, 6, piece.color):
                        moves.append((row, 6))  # Kingside castling
                
                # Queenside castling
                if (self.board[row][0] and 
                    self.board[row][0].piece_type == 'rook' and 
                    not self.board[row][0].has_moved and
                    not self.board[row][1] and 
                    not self.board[row][2] and 
                    not self.board[row][3]):
                    # Check if squares are not under attack
                    if not self.is_square_attacked(row, 3, piece.color) and not self.is_square_attacked(row, 2, piece.color):
                        moves.append((row, 2))  # Queenside castling
        
        # Filter out moves that would put/leave the king in check
        valid_moves = []
        for move in moves:
            if not self.would_be_in_check(piece, move[0], move[1]):
                valid_moves.append(move)
        
        return valid_moves
    
    def find_king(self, color):
        for row in range(8):
            for col in range(8):
                piece = self.board[row][col]
                if piece and piece.color == color and piece.piece_type == 'king':
                    return (row, col)
        return None
    
    def is_in_check(self, color):
        king_pos = self.find_king(color)
        if not king_pos:
            return False
        
        king_row, king_col = king_pos
        
        # Check if any opponent piece can attack the king
        opponent_color = 'black' if color == 'white' else 'white'
        for row in range(8):
            for col in range(8):
                piece = self.board[row][col]
                if piece and piece.color == opponent_color:
                    # Temporarily get all moves (even invalid ones to check attacks)
                    moves = self.get_all_attacking_moves(piece)
                    if (king_row, king_col) in moves:
                        return True
        return False
    
    def is_square_attacked(self, row, col, color):
        # Check if a square is attacked by any opponent piece
        opponent_color = 'black' if color == 'white' else 'white'
        for r in range(8):
            for c in range(8):
                piece = self.board[r][c]
                if piece and piece.color == opponent_color:
                    moves = self.get_all_attacking_moves(piece)
                    if (row, col) in moves:
                        return True
        return False
    
    def get_all_attacking_moves(self, piece):
        # Get all possible moves without checking for check, but pawns only attack diagonally
        moves = []
        row, col = piece.row, piece.col
        
        if piece.piece_type == 'pawn':
            direction = -1 if piece.color == 'white' else 1
            
            # Pawns attack diagonally only
            for dc in [-1, 1]:
                r, c = row + direction, col + dc
                if 0 <= r < 8 and 0 <= c < 8:
                    moves.append((r, c))
        
        elif piece.piece_type == 'rook':
            # Horizontal and vertical moves
            for dr, dc in [(0, 1), (1, 0), (0, -1), (-1, 0)]:
                for i in range(1, 8):
                    r, c = row + i * dr, col + i * dc
                    if not (0 <= r < 8 and 0 <= c < 8):
                        break
                    moves.append((r, c))
                    if self.board[r][c]:
                        break
        
        elif piece.piece_type == 'knight':
            # L-shaped moves
            for dr, dc in [(2, 1), (1, 2), (-1, 2), (-2, 1), (-2, -1), (-1, -2), (1, -2), (2, -1)]:
                r, c = row + dr, col + dc
                if 0 <= r < 8 and 0 <= c < 8:
                    moves.append((r, c))
        
        elif piece.piece_type == 'bishop':
            # Diagonal moves
            for dr, dc in [(1, 1), (1, -1), (-1, 1), (-1, -1)]:
                for i in range(1, 8):
                    r, c = row + i * dr, col + i * dc
                    if not (0 <= r < 8 and 0 <= c < 8):
                        break
                    moves.append((r, c))
                    if self.board[r][c]:
                        break
        
        elif piece.piece_type == 'queen':
            # Combination of rook and bishop moves
            for dr, dc in [(0, 1), (1, 0), (0, -1), (-1, 0), (1, 1), (1, -1), (-1, 1), (-1, -1)]:
                for i in range(1, 8):
                    r, c = row + i * dr, col + i * dc
                    if not (0 <= r < 8 and 0 <= c < 8):
                        break
                    moves.append((r, c))
                    if self.board[r][c]:
                        break
        
        elif piece.piece_type == 'king':
            # One square in any direction
            for dr in [-1, 0, 1]:
                for dc in [-1, 0, 1]:
                    if dr == 0 and dc == 0:
                        continue
                    r, c = row + dr, col + dc
                    if 0 <= r < 8 and 0 <= c < 8:
                        moves.append((r, c))
        
        return moves
    
    def would_be_in_check(self, piece, to_row, to_col):
        # Temporarily make the move
        from_row, from_col = piece.row, piece.col
        captured_piece = self.board[to_row][to_col]
        
        self.board[from_row][from_col] = None
        self.board[to_row][to_col] = piece
        piece.row, piece.col = to_row, to_col
        
        # Check if king is in check
        in_check = self.is_in_check(piece.color)
        
        # Undo the move
        self.board[from_row][from_col] = piece
        self.board[to_row][to_col] = captured_piece
        piece.row, piece.col = from_row, from_col
        
        return in_check
    
    def is_checkmate(self, color):
        if not self.is_in_check(color):
            return False
        
        # Check if any move can get out of check
        for row in range(8):
            for col in range(8):
                piece = self.board[row][col]
                if piece and piece.color == color:
                    moves = self.get_valid_moves(piece)
                    if moves:
                        return False
        return True
    
    def is_stalemate(self, color):
        if self.is_in_check(color):
            return False
        
        # Check if any move is possible
        for row in range(8):
            for col in range(8):
                piece = self.board[row][col]
                if piece and piece.color == color:
                    moves = self.get_valid_moves(piece)
                    if moves:
                        return False
        return True
    
    def reset(self):
        self.__init__()
    
    def start_server(self, port, side):
        try:
            self.server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            self.server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
            self.server_socket.bind(('localhost', port))
            self.server_socket.listen(1)
            print(f"Server listening on port {port}")
            self.connection, addr = self.server_socket.accept()
            print(f"Connected to {addr}")
            self.network_mode = 'server'
            self.player_side = side
            self.is_my_turn = (side == 'white')
            self.view_rotated = (side == 'black')
            self.move_count = 0
            self.game_mode = 'playing'
            # Send player side to client
            self.send_move({'type': 'side', 'side': 'black' if side == 'white' else 'white'})
            # Start receiving moves in a separate thread
            threading.Thread(target=self.receive_moves, daemon=True).start()
        except Exception as e:
            print(f"Server error: {e}")
    
    def connect_to_server(self, host, port):
        try:
            self.client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            self.client_socket.connect((host, port))
            print(f"Connected to server at {host}:{port}")
            self.connection = self.client_socket
            self.network_mode = 'client'
            self.move_count = 0
            # Wait for side assignment from server
            self.game_mode = 'playing'
            # Start receiving moves in a separate thread
            threading.Thread(target=self.receive_moves, daemon=True).start()
        except Exception as e:
            print(f"Client error: {e}")
    
    def send_move(self, move_data):
        if self.connection:
            try:
                message = json.dumps(move_data)
                self.connection.sendall(message.encode())
            except Exception as e:
                print(f"Send error: {e}")
                self.handle_connection_error()
    
    def receive_moves(self):
        while self.game_mode == 'playing':
            try:
                if self.connection:
                    data = self.connection.recv(1024)
                    if not data:
                        self.handle_connection_error()
                        break
                    move_data = json.loads(data.decode())
                    if move_data.get('type') == 'side':
                        self.player_side = move_data['side']
                        self.is_my_turn = (self.player_side == 'white')
                        self.view_rotated = (self.player_side == 'black')
                    else:
                        self.apply_remote_move(move_data)
            except Exception as e:
                print(f"Receive error: {e}")
                self.handle_connection_error()
                break
            time.sleep(0.1)
    
    def handle_connection_error(self):
        print("Connection lost")
        self.game_mode = 'menu'
        if self.server_socket:
            self.server_socket.close()
            self.server_socket = None
        if self.client_socket:
            self.client_socket.close()
            self.client_socket = None
        if self.connection:
            self.connection.close()
            self.connection = None
    
    def apply_remote_move(self, move_data):
        from_pos = move_data['from']
        to_pos = move_data['to']
        
        # Move the piece
        piece = self.board[from_pos[0]][from_pos[1]]
        self.board[from_pos[0]][from_pos[1]] = None
        self.board[to_pos[0]][to_pos[1]] = piece
        piece.row, piece.col = to_pos
        piece.has_moved = True
        
        # Handle special moves
        if move_data.get('castling'):
            # Move the rook
            if to_pos[1] > from_pos[1]:  # Kingside
                rook = self.board[from_pos[0]][7]
                self.board[from_pos[0]][7] = None
                self.board[from_pos[0]][5] = rook
                rook.col = 5
                rook.has_moved = True
            else:  # Queenside
                rook = self.board[from_pos[0]][0]
                self.board[from_pos[0]][0] = None
                self.board[from_pos[0]][3] = rook
                rook.col = 3
                rook.has_moved = True
        
        if move_data.get('en_passant'):
            # Remove captured pawn
            captured_pawn_row = from_pos[0]
            captured_pawn_col = to_pos[1]
            self.board[captured_pawn_row][captured_pawn_col] = None
        
        # Switch turns
        self.current_player = 'black' if self.current_player == 'white' else 'white'
        self.move_count += 1
        self.is_my_turn = not self.is_my_turn
        
        # Check for check/checkmate
        self.check = self.is_in_check(self.current_player)
        if self.check:
            if self.is_checkmate(self.current_player):
                self.checkmate = True
                self.game_over = True
                self.winner = 'black' if self.current_player == 'white' else 'white'
            elif self.is_stalemate(self.current_player):
                self.stalemate = True
                self.game_over = True
        elif self.is_stalemate(self.current_player):
            self.stalemate = True
            self.game_over = True

# Create game instance
game = ChessGame()

# Create buttons for menu
local_button = Button(WIDTH//2 - 100, HEIGHT//2 - 50, 200, 50, "Local Play")
online_button = Button(WIDTH//2 - 100, HEIGHT//2 + 20, 200, 50, "Play Online")
quit_button = Button(WIDTH//2 - 100, HEIGHT//2 + 90, 200, 50, "Quit")

# Create buttons for online menu
server_button = Button(WIDTH//2 - 100, HEIGHT//2 - 50, 200, 50, "Host Game")
client_button = Button(WIDTH//2 - 100, HEIGHT//2 + 20, 200, 50, "Join Game")
back_button = Button(WIDTH//2 - 100, HEIGHT//2 + 160, 200, 50, "Back")

# Create buttons for server side selection
white_side_button = Button(WIDTH//2 - 100, HEIGHT//2 - 50, 200, 50, "Play as White")
black_side_button = Button(WIDTH//2 - 100, HEIGHT//2 + 20, 200, 50, "Play as Black")

# Create input boxes for online setup
port_input = InputBox(WIDTH//2 - 100, HEIGHT//2 - 30, 200, 40, "5555")
host_input = InputBox(WIDTH//2 - 100, HEIGHT//2 - 30, 200, 40, "localhost")

# Main game loop
clock = pygame.time.Clock()
running = True

while running:
    mouse_pos = pygame.mouse.get_pos()
    
    # Handle events
    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            running = False
        elif event.type == pygame.KEYDOWN:
            if event.key == pygame.K_r and game.game_mode == 'playing':
                game.reset()
                game.game_mode = 'menu'
            elif event.key == pygame.K_ESCAPE and game.game_mode in ['server_setup', 'client_setup', 'side_selection']:
                if game.game_mode == 'side_selection':
                    game.game_mode = 'online_menu'
                else:
                    game.game_mode = 'online_menu'
        elif event.type == pygame.MOUSEBUTTONDOWN:
            if game.game_mode == 'menu':
                if local_button.is_clicked(mouse_pos, event):
                    game.game_mode = 'playing'
                    game.player_side = None
                elif online_button.is_clicked(mouse_pos, event):
                    game.game_mode = 'online_menu'
                elif quit_button.is_clicked(mouse_pos, event):
                    running = False
            elif game.game_mode == 'online_menu':
                if server_button.is_clicked(mouse_pos, event):
                    game.game_mode = 'side_selection'
                elif client_button.is_clicked(mouse_pos, event):
                    game.game_mode = 'client_setup'
                elif back_button.is_clicked(mouse_pos, event):
                    game.game_mode = 'menu'
            elif game.game_mode == 'side_selection':
                if white_side_button.is_clicked(mouse_pos, event):
                    game.game_mode = 'server_setup'
                    game.player_side = 'white'
                elif black_side_button.is_clicked(mouse_pos, event):
                    game.game_mode = 'server_setup'
                    game.player_side = 'black'
            elif game.game_mode == 'server_setup':
                port_input.handle_event(event)
                if event.type == pygame.MOUSEBUTTONDOWN:
                    if WIDTH//2 - 100 <= mouse_pos[0] <= WIDTH//2 + 100 and HEIGHT//2 + 50 <= mouse_pos[1] <= HEIGHT//2 + 100:
                        try:
                            port = int(port_input.text)
                            game.start_server(port, game.player_side)
                        except ValueError:
                            print("Invalid port number")
            elif game.game_mode == 'client_setup':
                host_input.handle_event(event)
                port_input.handle_event(event)
                if event.type == pygame.MOUSEBUTTONDOWN:
                    if WIDTH//2 - 100 <= mouse_pos[0] <= WIDTH//2 + 100 and HEIGHT//2 + 80 <= mouse_pos[1] <= HEIGHT//2 + 130:
                        try:
                            port = int(port_input.text)
                            game.connect_to_server(host_input.text, port)
                        except ValueError:
                            print("Invalid port number")
            elif game.game_mode == 'playing':
                if event.button == 1:  # Left mouse button
                    # Get board position accounting for rotation
                    row, col = game.get_board_position(event.pos[0], event.pos[1])
                    
                    if 0 <= row < 8 and 0 <= col < 8:
                        # If it's a human player's turn or local play
                        if (game.player_side is None or 
                            (game.player_side == game.current_player and game.is_my_turn)):
                            if game.selected_piece:
                                # Try to move the selected piece
                                if not game.move_piece(row, col):
                                    # If move failed, try to select a new piece
                                    game.select_piece(row, col)
                                else:
                                    # If move succeeded and in network mode, send move
                                    if game.network_mode:
                                        move_data = {
                                            'from': (game.move_history[-1]['from'][0], game.move_history[-1]['from'][1]),
                                            'to': (game.move_history[-1]['to'][0], game.move_history[-1]['to'][1]),
                                            'castling': game.move_history[-1]['castling'],
                                            'en_passant': game.move_history[-1]['en_passant']
                                        }
                                        game.send_move(move_data)
                            else:
                                # Select a piece
                                game.select_piece(row, col)
    
    # Update button hover states
    if game.game_mode == 'menu':
        local_button.check_hover(mouse_pos)
        online_button.check_hover(mouse_pos)
        quit_button.check_hover(mouse_pos)
    elif game.game_mode == 'online_menu':
        server_button.check_hover(mouse_pos)
        client_button.check_hover(mouse_pos)
        back_button.check_hover(mouse_pos)
    elif game.game_mode == 'side_selection':
        white_side_button.check_hover(mouse_pos)
        black_side_button.check_hover(mouse_pos)
    elif game.game_mode == 'server_setup':
        port_input.update()
    elif game.game_mode == 'client_setup':
        host_input.update()
        port_input.update()
    
    # Draw everything
    screen.fill((50, 50, 50))  # Background
    
    if game.game_mode == 'menu':
        # Draw title
        title = title_font.render("Chess Game", True, WHITE)
        screen.blit(title, (WIDTH//2 - title.get_width()//2, 100))
        
        # Draw buttons
        local_button.draw(screen)
        online_button.draw(screen)
        quit_button.draw(screen)
        
        # Draw instructions
        instructions = small_font.render("Choose your game mode", True, WHITE)
        screen.blit(instructions, (WIDTH//2 - instructions.get_width()//2, HEIGHT - 50))
    elif game.game_mode == 'online_menu':
        # Draw title
        title = title_font.render("Online Play", True, WHITE)
        screen.blit(title, (WIDTH//2 - title.get_width()//2, 100))
        
        # Draw buttons
        server_button.draw(screen)
        client_button.draw(screen)
        back_button.draw(screen)
    elif game.game_mode == 'side_selection':
        # Draw title
        title = title_font.render("Choose Your Side", True, WHITE)
        screen.blit(title, (WIDTH//2 - title.get_width()//2, 100))
        
        # Draw buttons
        white_side_button.draw(screen)
        black_side_button.draw(screen)
        
        # Draw back instruction
        back_text = small_font.render("Press ESC to go back", True, WHITE)
        screen.blit(back_text, (WIDTH//2 - back_text.get_width()//2, HEIGHT - 50))
    elif game.game_mode == 'server_setup':
        # Draw title
        title = title_font.render("Host Game", True, WHITE)
        screen.blit(title, (WIDTH//2 - title.get_width()//2, 100))
        
        # Draw instructions
        instructions = font.render("Enter port number:", True, WHITE)
        screen.blit(instructions, (WIDTH//2 - instructions.get_width()//2, HEIGHT//2 - 100))
        
        # Draw input box
        port_input.draw(screen)
        
        # Draw start button
        pygame.draw.rect(screen, BUTTON_COLOR, (WIDTH//2 - 100, HEIGHT//2 + 50, 200, 50), border_radius=10)
        pygame.draw.rect(screen, WHITE, (WIDTH//2 - 100, HEIGHT//2 + 50, 200, 50), 2, border_radius=10)
        start_text = font.render("Start Server", True, WHITE)
        screen.blit(start_text, (WIDTH//2 - start_text.get_width()//2, HEIGHT//2 + 60))
        
        # Draw back instruction
        back_text = small_font.render("Press ESC to go back", True, WHITE)
        screen.blit(back_text, (WIDTH//2 - back_text.get_width()//2, HEIGHT - 50))
    elif game.game_mode == 'client_setup':
        # Draw title
        title = title_font.render("Join Game", True, WHITE)
        screen.blit(title, (WIDTH//2 - title.get_width()//2, 50))
        
        # Draw host instructions
        host_label = font.render("Host IP:", True, WHITE)
        screen.blit(host_label, (WIDTH//2 - host_label.get_width()//2, HEIGHT//2 - 120))
        host_input.draw(screen)
        
        # Draw port instructions
        port_label = font.render("Port:", True, WHITE)
        screen.blit(port_label, (WIDTH//2 - port_label.get_width()//2, HEIGHT//2 - 30))
        port_input.draw(screen)
        
        # Draw connect button
        pygame.draw.rect(screen, BUTTON_COLOR, (WIDTH//2 - 100, HEIGHT//2 + 80, 200, 50), border_radius=10)
        pygame.draw.rect(screen, WHITE, (WIDTH//2 - 100, HEIGHT//2 + 80, 200, 50), 2, border_radius=10)
        connect_text = font.render("Connect", True, WHITE)
        screen.blit(connect_text, (WIDTH//2 - connect_text.get_width()//2, HEIGHT//2 + 90))
        
        # Draw back instruction
        back_text = small_font.render("Press ESC to go back", True, WHITE)
        screen.blit(back_text, (WIDTH//2 - back_text.get_width()//2, HEIGHT - 50))
    elif game.game_mode == 'playing':
        game.draw_board(screen)
        
        # Draw game status
        status_text = ""
        if game.game_over:
            if game.checkmate:
                winner = "White" if game.winner == 'white' else "Black"
                status_text = f"Checkmate! {winner} wins!"
            elif game.stalemate:
                status_text = "Stalemate! It's a draw."
        elif game.check:
            player = "White" if game.current_player == 'white' else "Black"
            status_text = f"{player} is in check!"
        else:
            player = "White" if game.current_player == 'white' else "Black"
            if game.player_side:
                if game.network_mode:
                    if game.is_my_turn:
                        status_text = f"Your turn ({player})"
                    else:
                        status_text = f"Opponent's turn ({player})"
                else:
                    if game.player_side == game.current_player:
                        status_text = f"Your turn ({player})"
                    else:
                        status_text = f"Opponent's turn ({player})"
            else:
                status_text = f"{player}'s turn"
        
        text = font.render(status_text, True, WHITE)
        screen.blit(text, (WIDTH // 2 - text.get_width() // 2, 10))
        
        # Draw move count for debugging
        move_count_text = small_font.render(f"Move: {game.move_count}", True, WHITE)
        screen.blit(move_count_text, (10, 10))
        
        # Draw instructions
        if game.network_mode:
            if game.is_my_turn:
                instructions = small_font.render("Your turn - click to select/move pieces", True, WHITE)
            else:
                instructions = small_font.render("Waiting for opponent...", True, WHITE)
        elif game.player_side:
            instructions = small_font.render("You are playing as " + game.player_side.capitalize(), True, WHITE)
        else:
            instructions = small_font.render("Local play - click to select/move pieces", True, WHITE)
        screen.blit(instructions, (WIDTH // 2 - instructions.get_width() // 2, HEIGHT - 30))
    
    # Update display
    pygame.display.flip()
    
    # Cap the frame rate
    clock.tick(60)

# Clean up sockets
if game.server_socket:
    game.server_socket.close()
if game.client_socket:
    game.client_socket.close()
if game.connection:
    game.connection.close()

# Quit pygame
pygame.quit()
sys.exit()
